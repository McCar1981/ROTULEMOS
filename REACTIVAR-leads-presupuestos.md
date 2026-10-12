# Runbook — Reactivar el procesamiento automático de presupuestos y leads del Asesor IA

> **Dónde ejecutar esto:** en tu **Claude Code LOCAL** (el que tiene acceso SSH/SFTP al servidor de dondominio), **no** en la sesión web. La web solo llega por HTTPS; no alcanza el buzón `info@rotulemos.com` (IMAP) ni el cron del servidor.
>
> Este documento lo preparó la sesión web tras ingeniería inversa del sitio en vivo. Todo lo marcado como **VERIFICADO** está comprobado contra el motor real al céntimo: úsalo tal cual, no lo reinventes.

---

## 1. Objetivo

Reactivar, **en modo borrador**, el procesamiento automático de:
- **Solicitudes de presupuesto** entrantes a `info@rotulemos.com`.
- **Leads del Asesor IA** (correos titulados **`NUEVO LEAD ASESOR IA`**).

Con tres condiciones:
1. Vaciar el **backlog** acumulado (hay varios sin responder), no solo los nuevos.
2. **Todos los precios salen del motor real** de la web. La IA nunca escribe una cifra.
3. Arreglar el **bug del Asesor IA** (los precios salen en blanco en el chat).

---

## 2. Diagnóstico de arranque (comprobar en el servidor por SSH)

- **Cron:** `crontab -l`, `/etc/cron.d/`, `/etc/crontab`. Buscar el script que lee el buzón cada 5-10 min. ¿Está presente, activo, o comentado?
- **Script lector / "librería":** la sesión anterior terminó en *"escribo la librería"* → puede estar a medio terminar. `grep -rl -e imap_open -e "NUEVO LEAD" -e recalcular_resumen -e presupuesto` en el directorio del sitio.
- **Por qué está parado** (una de estas): cron desactivado · credencial IMAP de `info@rotulemos.com` caducada · API key de Anthropic (ya tienes saldo en Console — verificar que la key del `.env`/config del servidor es la correcta) · script sin terminar.
- **Registro de procesados:** localizar la tabla/fichero donde se marca cada correo ya respondido (para no duplicar y para saber qué queda de backlog). `bd_conexion.php` apunta a la BD.

---

## 3. Contrato del motor de precios — **VERIFICADO**

El único cálculo válido. Es el que usa el configurador de la ficha.

- **Endpoint:** `POST https://www.rotulemos.com/ajax/recalcular_resumen.php`
- **Headers:** `X-Requested-With: XMLHttpRequest`, `Content-Type: application/x-www-form-urlencoded; charset=UTF-8`, `Referer: <url de la ficha del producto>`.
- **Payload** (serialización estilo jQuery `$.param`, claves exactas):
  - `dimensiones[alto]`, `dimensiones[ancho]` (cm)
  - `fecha` = id de plazo → **`3` = Standard, `2` = Rápido, `1` = Express**
  - `acc[i][acc]` = **un acabado por sección** (el `data-acc` de la opción elegida). Las secciones se agrupan por `data-familia` en la ficha; se manda el seleccionado de cada una.
  - `idprod`, `unidades`
  - campos que van **vacíos** en lonas/vinilos: `letrasCorp`, `colorRal`, `factorComplejidad`, `perimetro`, `altoRigidosOrig`, `anchoRigidosOrig`
- **Respuesta JSON:** `{ "coste": <float>, "iva": ..., "area": ... }` → **`coste` es el precio FINAL con IVA**. El importe sin IVA = `coste / 1.21`.
- **Validación obligatoria antes de fiarte:** reproduce un presupuesto conocido. Ej. **Lonas de obra** (`idprod 88`), 290×170 cm, 9 ud, `acc = [394, 396, 392, 399]`, `fecha=3` → **348,98 €** (coincide al céntimo con el presupuesto real del 25-sep).

### 3.1 Script Python listo para calcular/validar (reutilizable)

```python
import subprocess, urllib.parse, json

def jq(p, o, out):
    if isinstance(o, list):
        for i, v in enumerate(o):
            jq(f"{p}[{i}]" if isinstance(v,(list,dict)) else f"{p}[]", v, out)
    elif isinstance(o, dict):
        for k, v in o.items():
            jq(f"{p}[{k}]" if p else k, v, out)
    else:
        out.append((p, '' if o is None else str(o)))

def precio(idprod, ancho, alto, unidades, accs, fecha, referer):
    """accs = lista de data-acc (uno por seccion). Devuelve coste con IVA."""
    data = {
        'dimensiones': {'alto': alto, 'ancho': ancho}, 'fecha': fecha,
        'acc': [{'acc': a, 'ladosSel': []} for a in accs],
        'idprod': idprod, 'unidades': unidades,
        'letrasCorp': None, 'colorRal': None, 'factorComplejidad': None,
        'perimetro': None, 'altoRigidosOrig': None, 'anchoRigidosOrig': None,
    }
    out = []; [jq(k, v, out) for k, v in data.items()]
    body = '&'.join(f"{urllib.parse.quote_plus(k)}={urllib.parse.quote_plus(v)}" for k, v in out)
    r = subprocess.run(['curl','-sS','-X','POST',
        'https://www.rotulemos.com/ajax/recalcular_resumen.php',
        '-H','X-Requested-With: XMLHttpRequest',
        '-H','Content-Type: application/x-www-form-urlencoded; charset=UTF-8',
        '-H', f'Referer: {referer}', '-H','User-Agent: Mozilla/5.0',
        '--data', body], capture_output=True, text=True, timeout=40)
    return json.loads(r.stdout.strip())['coste']

# Validacion:
# precio(88, 290, 170, 9, ['394','396','392','399'], 3,
#        'https://www.rotulemos.com/producto/Lonas-para-obras')  -> 348.98
```

### 3.2 IDs ya extraídos — **VERIFICADO**

| Producto | idprod | Notas de acabados |
|---|---|---|
| Lonas para obras | **88** | Material 394 frontlit / 395 microperf · Impresión 396 color / 397 sin · **Refuerzo perimetral 392 No / 393 Sí (cuesta 0 €, incluido)** · Ojales 398 sin / **399 metálicos (+53,69 €)** |
| Vinilo Polimérico Premium | **70** | Defaults por familia: `{111:370, 113:372, 110:367, 130:408, 115:376, 76:280, 114:374, 112:371}`. Familia 76 = Plastificado/Laminado → **Sin laminado 280 · Polimérico Mate+UV 403 · Polimérico Brillo+UV 404 (403=404 mismo precio)** · Monomérico 281/282 · Antigraffiti 407 |
| Vinilo fundición vehículos | **97** | Se tarifa por m² (≈ 22 €/m² + IVA, mínimo ~1 m²) |

Para cualquier otro producto, mismo método: `GET` la ficha, sacar `idProd = 'NN'` del JS y los `data-acc`/`data-familia` de las secciones `sec-acabados`.

---

## 4. Bug del Asesor IA — **arreglar**

En las conversaciones del chat los precios salen **en blanco** (ej. lead de Raquel: todas las medidas sin número). Causa conocida: el asesor calcula con una **copia simplificada** del cálculo (aprox. `chat-rotulemos.php` ~línea 691 según la sesión anterior), sin topes ni acabados, y/o no renderiza.

- El asesor debe llamar al **mismo** `recalcular_resumen.php` que la ficha, nunca a una copia.
- Y el **correo de presupuesto** que se genera del lead debe **recalcular con el motor real**, no copiar lo que dijo el chat.

---

## 5. Reglas de clasificación de leads del Asesor IA

Leer la conversación que viene **dentro del propio correo del lead** y extraer: producto, medidas, unidades, acabados y destino (interior / exterior / suelo).

1. **Lead con producto + medidas + cantidad** → **presupuesto cerrado**: tabla Standard/Rápido/Express (IVA incl.), total si son varias piezas, envío a península incluido. *(Ej. Raquel: vinilo polimérico premium, 3 medidas, laminado UV.)*
2. **Lead con producto pero SIN medidas** → **correo de re-enganche**: resolver la duda + **orientación de precio real** (ej. "desde 22 €/m² + IVA") + CTA a configurar en la web. *(Ej. jroviracristia: "tengo una duda" sobre vinilo fundición.)*
3. Elegir el acabado correcto según el uso: exterior → material polimérico + **laminado polimérico con filtro UV** (no monomérico).

---

## 5B. Qué NO se auto-presupuesta — **gate crítico**

No todo correo de "presupuesto" es un producto de catálogo. **Si el encargo no es configurable en la web, el sistema NO debe auto-presupuestar: lo marca para revisión humana** y, como mucho, manda un acuse ("hemos recibido su encargo; un especialista le prepara un presupuesto a medida"). Auto-cotizar el producto de catálogo más parecido es **peor que no contestar**: hace quedar mal ante un buen cliente.

Señales de "a mano, no auto" (si aparece alguna → humano):
- **Iluminación / LED / retroiluminado**, letras **corpóreas / recortadas / opacas**, **banderola luminosa**.
- **Anclajes, estructura, instalación**, fachada **protegida / normativa urbanística**.
- Varias piezas que forman un **rótulo compuesto**, o acabados/materiales que no están en el configurador.

**Caso real (no repetir el error):** despacho de abogados en Aranjuez (Juan Pablo González Rivas). Pidió un **rótulo de letras opacas retroiluminadas sobre soporte transparente + banderola luminosa**. El sistema le auto-cotizó **placas de metacrilato impreso CMYK** (catálogo) y **omitió la iluminación y la banderola** —lo primero que pedía—. Cliente de calidad, casi perdido por una respuesta automática que no leía su encargo. Esto es justo lo que el gate debe cortar.

---

## 6. CTA directo al carrito

Los correos llevan botones **"Pedir Standard/Rápido/Express"** que abren la web **directamente en el carrito ya configurado**. Mecánica real:
- El carrito (`/carro`) es **de sesión en el servidor**; `ajax/addToCart.php` es **solo POST** (GET da 403).
- Por tanto el CTA es un **endpoint GET a medida** que monta el carrito y redirige a `/carro`. **Está en tu servidor** (lo creó la sesión anterior; es el generador del correo del 25-sep). Reutilízalo.
- **Verificar** que el enlace codifica **todos** los acabados elegidos (ej. `refuerzo perimetral 393`, `laminado UV 403`) para que producción reciba la pieza correcta, no los defaults.

---

## 7. Backlog y modo borrador

- Procesar **todo lo acumulado** (`NUEVO LEAD ASESOR IA` + solicitudes de presupuesto), marcando cada uno como procesado para no duplicar.
- **Modo borrador** 2-3 semanas: cada respuesta a **carpeta Borradores** + tarjeta en **Trello** para revisar y enviar a mano. Validar que los precios son buenos antes de activar envío automático (de momento solo casos simples sin instalación; instalación **solo Valencia y alrededores**).

---

## 8. Prompt para pegar en tu Claude Code local

```
Reactiva el sistema de presupuestos automáticos de Rotulemos en el servidor (SSH/SFTP dondominio), en modo borrador. Sigue el runbook REACTIVAR-leads-presupuestos.md que te paso. Pasos:
1. Diagnostica por qué está parado el cron que lee info@rotulemos.com (sección 2).
2. Termina/arregla el lector de buzón y el generador de presupuesto si quedó a medias.
3. Añade el procesamiento de los correos "NUEVO LEAD ASESOR IA" con las reglas de la sección 5.
4. Asegura que TODO precio sale de recalcular_resumen.php (sección 3), nunca de una copia; arregla el bug del asesor (sección 4). Valida reproduciendo 348,98 € en lonas 88.
5. Reutiliza el generador de CTA-al-carrito existente y verifica que codifica los acabados elegidos (sección 6).
6. Procesa el backlog y deja todo en Borradores + Trello (sección 7).
Antes de tocar nada, enséñame el diagnóstico de la sección 2 y espera mi OK.
```

---

## 9. Casos ya resueltos a mano en esta sesión (listos para enviar hoy)

- **Conce Navas (Zaracla)** — 9 lonas 290×170 con refuerzo perimetral → **mismo precio** (el refuerzo va incluido, 0 €): **348,98 / 391,92 / 434,87 €** (IVA incl.). Texto de respuesta ya redactado.
- **Raquel Conesa** — Vinilo Polimérico Premium + laminado UV, 3 medidas (120×90, 200×70, 300×70, 1 ud c/u) → **94,77 / 104,74 / 114,72 €** (IVA incl.). Presupuesto ya redactado.
