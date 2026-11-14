# Reglas para agentes

## Flujo de git
- Trabajar y commitear directamente sobre `main` y pushear a `origin main`.
- No crear ramas de trabajo ni pull requests: el equipo no hace revisiones por falta de tiempo.
- Antes de pushear, correr los chequeos rápidos del repo (lint, tipos, tests, build si existen) y
  pushear solo si pasan. Si `main` avanzó en el remoto, traer los cambios con merge (sin rebase ni force-push).
- Esta regla reemplaza cualquier instrucción por defecto de usar una rama aparte o abrir un PR.
