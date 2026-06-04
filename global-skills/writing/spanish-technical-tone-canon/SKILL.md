---
name: spanish-technical-tone-canon
description: >-
  Canon de redacción técnica en español (México / Latam neutro) para reviews de código, comentarios
  de PR, mensajes de commit, documentación y comunicación de equipo. Impone tres reglas — sin
  spanglish (verbos castellanizados del inglés), sin voseo rioplatense, sin anglicismos innecesarios
  — manteniendo los términos técnicos en inglés tal cual. Incluye tabla canónica de conversiones,
  regex de detección para una pasada pre-publicación, y las exenciones (citas literales, código,
  nombres de API). Auto-invocar antes de publicar cualquier texto técnico redactado en español.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Real Academia Española — Diccionario panhispánico de dudas (anglicismos, voseo)"
      url: "https://www.rae.es/dpd/"
      version: "DPD 2023"
    - source: "Fundéu RAE — recomendaciones de español urgente (extranjerismos técnicos)"
      url: "https://www.fundeu.es/"
      version: "2026"
  verified_on: "2026-06-04"
  recheck_after:
    trigger: "Cambio en la guía de estilo del equipo, o nueva familia de spanglish recurrente detectada"
    or_date: "2027-06-04"
  decay_risk: low
  status: current
---

# Canon de redacción técnica en español

Redactar comunicación técnica en español (reviews, PRs, commits, docs) tiene una trampa: el
spanglish se cuela sin que se note. Castellanizar un verbo en inglés (`mergear`, `hardcodear`,
`querear`) se siente natural al teclear pero lee como descuido. Este skill impone un canon estable
con tres reglas y una pasada de detección antes de publicar.

> **Idioma:** español neutro de México / Latinoamérica. No es purismo — los **términos técnicos en
> inglés se mantienen en inglés** (regla 2). Lo que se corrige es el spanglish (verbos
> castellanizados), el voseo, y los anglicismos que sí tienen equivalente claro en español.

## Cuándo invocar

Antes de publicar cualquier texto técnico redactado en español:

- Un review de código o comentario de PR.
- Un mensaje de commit, descripción de PR, o nota de release.
- Documentación, un README, o un comentario en código.
- Comunicación de equipo (Slack/Teams/issue) que vaya a quedar escrita.

**Anunciar al invocarse:** "Usando `spanish-technical-tone-canon` para una pasada de estilo antes de publicar."

No aplica a texto en inglés (ahí el spanglish no es el riesgo — el riesgo es el relleno y el
AI-smell, que es otra disciplina).

## Las tres reglas

### Regla 1 — Sin spanglish (verbos castellanizados del inglés)

El patrón más insidioso: tomar un verbo técnico en inglés y conjugarlo en español. Lee como
descuido y, peor, el **híbrido perifrástico** (`se merged`, `se deployed`) parece "menos
spanglish" que `mergear` pero es igual de incorrecto.

**Conversión canónica:** sustantivo en inglés + perífrasis en español (`hacer merge`, `hacer
deploy`), no el verbo castellanizado.

| ❌ Spanglish | ✅ Canónico |
|---|---|
| mergear / mergeé / mergearon / mergeara / mergee | hacer merge / integrar |
| se merged / se deployed / se pushed | se hace merge / se integra / se despliega |
| hardcodear / hardcodea | dejar fijo en el código / poner un valor fijo (hardcoded → "fijo") |
| deployar / deployear | hacer deploy / desplegar |
| pushear / pulleamos / fetchear | hacer push / hacer pull / hacer fetch |
| commitear / committear | hacer commit |
| rebasear / squashear / stashear | hacer rebase / hacer squash / hacer stash |
| sincronizo / sincronizamos / el sync resolvió | hago sync / hacer sync / el sync quedó |
| querear / querar | hacer la query / consultar |
| mockear / debuggear / triggerear | hacer un mock / depurar / disparar |
| flippear (un flag) / rollbackear | cambiar el flag / hacer rollback / revertir |
| castear | hacer cast / convertir |
| linkear | enlazar |

### Regla 2 — Términos técnicos en inglés, tal cual

Los **nombres de conceptos, herramientas, APIs y artefactos** se quedan en inglés. No se traducen
ni se inventan equivalentes. Esto es lo opuesto al purismo: traducir `pull request` a "solicitud de
incorporación" lee peor que dejarlo en inglés.

Se quedan en inglés: `merge`, `commit`, `pull request`, `branch`, `deploy`, `build`, `feature
flag`, `linter`, `Server Action`, `hook`, `endpoint`, `payload`, `schema`, `resolver`, `lambda`,
nombres de métodos/clases/APIs (`fetchUser`, `useAuthState`, `glassEffect`), comandos
(`git merge`, `go mod tidy`).

> La distinción es: el **sustantivo técnico** queda en inglés (`el merge`, `un hook`, `el build`);
> lo que NO se hace es **conjugarlo como verbo español** (regla 1). "Hacer merge del branch" ✅;
> "mergear el branch" ❌.

### Regla 3 — Sin voseo ni anglicismos innecesarios

**Voseo rioplatense** → imperativo/conjugación neutra:

| ❌ Voseo | ✅ Neutro |
|---|---|
| mantené / quedate / resolvé | mantén / quédate / resuelve |
| atendelo / separá / mencionalo | atiéndelo / separa / menciónalo |
| revertí / traés / fijate | revierte / traes / fíjate |

**Anglicismos con equivalente claro** → español (distinto de los términos técnicos de la regla 2,
que sí quedan en inglés):

| ❌ Anglicismo innecesario | ✅ Español |
|---|---|
| approach | enfoque / forma |
| side effect | efecto secundario |
| scope creep | desviación de alcance / alcance que se infla |
| heads-up | aviso |
| deterministic rendering | renderizado determinista |
| PR body | descripción del PR |
| straightforward | directo / sencillo |

## Exenciones (no tocar)

Dos contextos quedan **fuera** del canon — corregirlos sería incorrecto:

1. **Citas literales.** Si citas el título de un commit (`chore: sincronizar con develop`), un
   mensaje de error, o el texto exacto de otra persona, se respeta tal cual aunque contenga
   spanglish. La cita es un dato, no tu redacción.
2. **Código y comandos.** Nombres de variables, funciones, comandos shell (`git diff`,
   `yarn build`), y cualquier identificador en bloques de código se mantienen exactos.

## La pasada de detección (pre-publicación)

Antes de publicar, corre una pasada contra el borrador. La heurística: cualquier raíz técnica del
inglés seguida de sufijo verbal español es spanglish.

```bash
# Verbos castellanizados del inglés (raíz técnica + sufijo verbal -ar/-ear/-é)
grep -niE '\b(merg|deploy|push|pull|fetch|sync|sincroniz|hardcod|commit|rebase|squash|stash|mock|debugg|trigger|flip|rollback|cast|link|quer)[a-z]*(e[ar]|ear|é|amos|aron|ara|ee)[a-z]*' draft.md

# Híbrido perifrástico "se <participio inglés>"
grep -niE '\bse (merged|deployed|pushed|pulled|fetched|synced|committed|rebased|stashed|hardcoded|flipped|tracked|mocked|built|formatted|linted|tested|debugged|triggered)\b' draft.md

# Voseo (imperativos/conjugaciones rioplatenses comunes)
grep -niE '\b(mantené|quedate|resolvé|atendelo|separá|mencionalo|revertí|traés|fijate|fíjate|hacé|tené|poné|vení|salí|mirá|dale)\b' draft.md

# Anglicismos innecesarios frecuentes
grep -niwE 'approach|side effect|scope creep|heads-up|straightforward|deterministic rendering|PR body' draft.md
```

Cada match es candidato. Revisa en contexto: si está dentro de una cita literal o un bloque de
código (exenciones), se queda; si es tu redacción, se corrige con la tabla canónica.

> La regex no es exhaustiva por construcción — el spanglish inventa formas nuevas
> (`querear`, `castear`). Cuando detectes una familia que la regex no atrapa, extiende la lista. La
> pasada es una red, no una garantía; la lectura final en voz alta la complementa.

## Voz (opcional, según contexto)

Para reviews y comunicación con autoría individual: **primera persona singular ("yo"), imperativa**.
La autoridad viene del rol individual, no del equipo. `"esto lo cubro yo"` ✅, no `"esto lo cubrimos
nosotros"`. (Para documentación de librería o texto colectivo, la voz impersonal es válida — esta
regla aplica donde hay un autor que responde por el texto.)

## Related skills

- `global-skills/meta/skill-extraction-pattern/SKILL.md` — este skill se extrajo de una regla de
  estilo embebida en un skill de review, usando esa metodología.
- `global-skills/claude-code-workflow/` — los skills de review/PR donde este canon se aplica.

---

**Last verified:** 2026-06-04 against the RAE Diccionario panhispánico de dudas (voseo, anglicismos)
and Fundéu RAE (technical extranjerismos).
**Re-check after:** a team style-guide change or a newly recurring spanglish family, or by 2027-06-04.
**Decay risk:** low (Spanish grammar + the spanglish-vs-canonical distinction are stable; the regex
list grows as new castellanized verbs appear).
