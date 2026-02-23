# Code Review Guidelines — Aprova Digital

Documento de referencia para reviews automatizados e manuais.

---

## Niveis de Severidade

| Nivel   | Label            | Bloqueia merge?          |
|---------|------------------|--------------------------|
| Critico | `[BLOQUEANTE]`   | Sim                      |
| Alto    | `[RECOMENDACAO]`  | Sim (se 3+ acumulados)   |
| Medio   | `[SUGESTAO]`      | Nao                      |
| Baixo   | `[NOTA]`          | Nao                      |

## Regras de Decisao

- **APPROVED**: Nenhum BLOQUEANTE e menos de 3 RECOMENDACOES
- **CHANGES REQUESTED**: 1+ BLOQUEANTE OU 3+ RECOMENDACOES
- **COMMENT**: Apenas SUGESTOES e NOTAS
- Falta de testes NUNCA bloqueia um PR — sugira como `[SUGESTAO]`

## Principios de Review

1. **Seja especifico e construtivo** — aponte o problema E sugira solucao com codigo
2. **Foque no codigo, nao na pessoa** — linguagem colaborativa ("considere", "podemos", "que tal")
3. **Reconheca boas praticas** — nao seja apenas critico
4. **Classifique cada comentario** com a severidade correspondente

---

## Checklist Backend

### BLOQUEANTE

| Item | Descricao |
|------|-----------|
| Dados sensiveis em logs | CPF, tokens, body completo, senhas, chaves de API em `console.log`, `logger.info`, etc. |
| Variaveis de modulo mutaveis | Variaveis `let` no escopo do modulo que sao modificadas em runtime — causam race conditions em Lambda/containers |
| Monkey-patching de globais | Sobrescrever `console.log`, `Date.now`, `Math.random` ou qualquer global do runtime |
| Projecao dinamica sem whitelist | `select` ou `include` construido a partir de input do usuario sem validacao de campos permitidos |
| Efeitos colaterais em dados de config | Mutar blueprints, configs, ou objetos compartilhados entre requests |
| Ambientes non-prod apontando para prod | URLs, connection strings, ou API keys de producao em ambientes de dev/staging |
| Secrets/credenciais hardcoded | Senhas, tokens, API keys diretamente no codigo-fonte |

### RECOMENDACAO

| Item | Descricao |
|------|-----------|
| `as any` em repositorios Prisma | Perda de type-safety — tipar corretamente com os types gerados pelo Prisma |
| `Promise<any>` como retorno de service | Retornos devem ser tipados para que consumers saibam o que esperar |
| Consumers SQS sem payload tipado | Mensagens de fila devem ter interface definida e validacao |
| Consumers SQS sem idempotencia | Reprocessamento de mensagens deve ser seguro (idempotent key) |
| Operacoes multi-entidade sem `$transaction` | Multiplas escritas no banco devem ser atomicas |
| Endpoints sem autenticacao explicita | Todo endpoint deve declarar explicitamente se e publico ou autenticado |
| Endpoints publicos sem rate limiting | Rotas publicas devem ter protecao contra abuso |
| `console.log` em producao | Usar logger estruturado (`@aprova-digital/logger` ou similar) |
| Retry sem backoff exponencial | Retries fixos podem causar thundering herd — usar exponential backoff |
| Erros silenciosos (catch vazio) | `catch {}` ou `catch (e) {}` sem log ou re-throw esconde problemas |
| Dead code e imports nao utilizados | Codigo morto aumenta complexidade cognitiva e tamanho do bundle |

---

## Checklist Frontend Angular

### RECOMENDACAO

| Item | Descricao |
|------|-----------|
| Subscriptions sem cleanup | Todo `subscribe()` deve ter cleanup via `takeUntil`, `DestroyableMixin`, `async pipe`, ou `takeUntilDestroyed()` |
| Nested subscriptions | `subscribe()` dentro de `subscribe()` — usar operadores RxJS (`switchMap`, `mergeMap`, `concatMap`) |
| `switchMap` em effects sem `catchError` | Erro nao tratado mata o effect permanentemente — envolver com `catchError` |
| `setTimeout` como hack de timing | Indica race condition com o DOM — usar `AfterViewInit`, `ChangeDetectorRef`, ou operadores RxJS |
| Componentes grandes (>300 linhas) | Dividir em componentes menores com responsabilidades claras |
| CSS duplicado sem extracao | Extrair para mixin, variavel CSS, ou componente compartilhado |
| Import completo de lodash | `import _ from 'lodash'` importa a lib inteira — usar `import { fn } from 'lodash-es'` |
| Dead code | Componentes, metodos, ou imports nao utilizados devem ser removidos |

### SUGESTAO

| Item | Descricao |
|------|-----------|
| Cores hex hardcoded | Usar CSS variables do Aprova Design System (ADS) — `var(--ads-color-*)` |
| `::ng-deep` e `!important` | Evitar quando possivel — usar encapsulamento adequado ou `ViewEncapsulation.None` com escopo |
| Constantes em camelCase | Constantes devem usar `UPPER_SNAKE_CASE` |
| Magic numbers | Numeros sem contexto devem ser extraidos para constantes nomeadas |

---

## Checklist Infraestrutura

### BLOQUEANTE

| Item | Descricao |
|------|-----------|
| Ambientes non-prod com config de prod | serverless.yml, terraform, ou env files com URLs/keys de producao em ambientes de dev/staging |
| Secrets hardcoded | Credenciais diretamente em arquivos de config — usar SSM, Secrets Manager, ou variaveis de CI |

### RECOMENDACAO

| Item | Descricao |
|------|-----------|
| Variaveis de ambiente sem fallback | Variaveis criticas devem ter validacao na inicializacao e fallback quando apropriado |

---

## Formato de Comentarios

Cada comentario de review DEVE:

1. Iniciar com o label de severidade: `[BLOQUEANTE]`, `[RECOMENDACAO]`, `[SUGESTAO]` ou `[NOTA]`
2. Descrever o problema de forma especifica (arquivo, linha)
3. Sugerir uma solucao com exemplo de codigo quando aplicavel
4. Ser construtivo e colaborativo

### Exemplo

```
[RECOMENDACAO] Subscription sem cleanup pode causar memory leak.

Considere usar `takeUntilDestroyed()` para cleanup automatico:

```typescript
// Antes
this.store.select(selectUser).subscribe(user => { ... });

// Depois
this.store.select(selectUser)
  .pipe(takeUntilDestroyed(this.destroyRef))
  .subscribe(user => { ... });
```
```

---

## Convencoes do Projeto

- PRs usam formato `[DK-XXXX] Descricao da mudanca`
- Commits em portugues, codigo em ingles
- Nao usar `console.log` em producao — usar logger estruturado
- Nao usar `as any` — tipar corretamente
- Libs compartilhadas: `@aprova-digital/shared`, `@aprova-digital/caramel-*`

## Stack

- **Backend**: Node.js 22, Fastify, Prisma (PostgreSQL), Inversify IoC, SQS, Memcached, Serverless/Lambda
- **Frontend**: Angular 15, NgRx, Module Federation, Aprova Design System (ADS)
- **Infra**: AWS (Lambda, ECS), Serverless Framework, Jenkins
