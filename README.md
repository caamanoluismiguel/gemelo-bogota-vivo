# Bogotwin: datos vivos

Este repo solo publica los datos que cambian cada pocos minutos de [Bogotwin](https://github.com/caamanoluismiguel/gemelo-bogota), un gemelo digital abierto e independiente de Bogotá,(https://github.com/caamanoluismiguel/gemelo-bogota). No guarda historia: cada 20 minutos una Action trae el código del repo principal, arma `cielo/ahora.json` y lo publica en GitHub Pages sin hacer commit.

- Datos: https://caamanoluismiguel.github.io/gemelo-bogota-vivo/cielo/ahora.json
- Código que los arma: `pipelines/cielo/cielo.py` en el repo principal.
- Cada bloque del JSON trae su fuente, la hora del dato, su licencia y si es medido o estimado. Las licencias son las de cada fuente (dominio público, CC BY 4.0, CC BY‑NC 4.0, CC BY‑SA 4.0), y lo NC y lo SA nunca se combinan en un mismo cálculo.
- Este proyecto es independiente y no es de la Alcaldía de Bogotá ni de sus entidades. No emite alertas: para eso están el IDIGER y el IDEAM.

Código bajo AGPL‑3.0.
