# randol_cityworker — Manual

Trabalho de eletricista da prefeitura: o jogador pega um veículo de serviço com o NPC, recebe pontos de reparo aleatórios no mapa e é pago a cada tarefa concluída.

---

## Sumário

1. [Dependências](#dependências)
2. [Instalação](#instalação)
3. [Configuração](#configuração)
4. [Fluxo do trabalho](#fluxo-do-trabalho)
5. [Anti-exploit](#anti-exploit)
6. [Integrações](#integrações)
7. [Estrutura de arquivos](#estrutura-de-arquivos)

---

## Dependências

| Recurso | Obrigatório | Observação |
|---|---|---|
| `qb-core` **ou** `es_extended` | Sim | Bridge automático em `bridge/`. O arquivo do framework inativo se desliga sozinho |
| `ox_lib` | Sim | `lib.require`, `lib.callback`, `lib.points`, `lib.progressCircle`, `lib.requestModel`, `lib.waitFor` |
| `qb-target` | Sim | Todas as interações (NPC e ponto de reparo) usam `exports['qb-target']` |
| `cw-rep` | Sim | `sv_cityworker.lua` chama `exports["cw-rep"]:updateSkill(src, "cityworker", 5)` a cada pagamento, sem checagem de existência |
| `oxmysql` | Não | Só é tocado pelo banimento anti-exploit do bridge QB (`INSERT INTO bans`). O `fxmanifest.lua` não carrega `@oxmysql/lib/MySQL.lua`, então essa chamada depende do `MySQL` global estar disponível |
| Script de combustível | Não | Só se `FuelScript.enable = true` |

---

## Instalação

1. Copie a pasta `randol_cityworker` para `resources/`.
2. Adicione ao `server.cfg`:
   ```
   ensure randol_cityworker
   ```
3. Garanta que o `cw-rep` esteja iniciado antes deste recurso — o pagamento chama o export dele.
4. Não há SQL próprio nem itens de inventário a cadastrar.

---

## Configuração

### `config.lua` (compartilhado)

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `FuelScript.enable` | bool | Sim | `true` usa o export do script de combustível; `false` seta `Entity(veh).state.fuel = 100` |
| `FuelScript.script` | string | Sim | Nome do recurso de combustível chamado via `exports[script]:SetFuel(veh, 100.0)` |
| `BossModel` | hash | Sim | Modelo do NPC que dá o trabalho (padrão `s_m_y_construct_02`) |
| `BossCoords` | `vec4` | Sim | Posição e heading do NPC. Também é o ponto usado para bater o ponto de saída (raio de 10.0) e a origem do blip fixo |

### `sv_config.lua` (servidor)

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `Timeout` | número (ms) | Sim | Espera antes de gerar a próxima tarefa após um pagamento (padrão `5000`) |
| `Account` | string | Sim | Conta usada no `AddMoney` (padrão `cash`; no ESX o bridge converte `cash` para `money`) |
| `Payout.min` / `Payout.max` | número | Sim | Faixa de pagamento sorteada por tarefa |
| `Locations` | lista de `vec3` | Sim | Pontos de reparo. Um é sorteado por tarefa (20 pontos por padrão) |
| `Vehicle` | hash | Sim | Modelo do veículo de trabalho (padrão `bison`) |
| `VehicleSpawn` | `vec4` | Sim | Onde o veículo é criado; o jogador é warpado direto para o banco do motorista |

---

## Fluxo do trabalho

1. Um blip fixo (sprite 566, "Central do Eletricista") marca `BossCoords`. O NPC só é criado quando o jogador entra num raio de 50 metros.
2. Opção de target **"Começar a trabalhar"** — o servidor cria o veículo, sorteia a primeira tarefa e devolve o `netid`. O cliente aplica placa `CITY####`, cor 111 e combustível cheio.
3. A tarefa vira um blip com rota e uma zona circular de 1.3 de raio no ponto sorteado, com a opção de target **"Reparar"**.
4. O reparo é um `lib.progressCircle` de 10 segundos com o scenario `WORLD_HUMAN_HAMMERING`. Ao concluir, o servidor paga, credita XP no `cw-rep` e, após `Timeout`, envia a próxima tarefa automaticamente.
5. Opção de target **"Terminar o trabalho"** no NPC — só funciona a menos de 10 metros dele. O veículo de trabalho é deletado e o turno encerra.

O veículo também é deletado quando o jogador cai do servidor (`playerDropped`) ou desloga do personagem.

---

## Anti-exploit

O pagamento é validado no servidor: se o jogador não tiver tarefa ativa ou estiver a mais de 10 metros do ponto de reparo, `handleExploit` é chamado.

- **QB** — insere um ban permanente na tabela `bans` (`bannedby = 'randol_cityworker'`) e derruba o jogador.
- **ESX** — apenas derruba o jogador e imprime um aviso no console.

---

## Integrações

### cw-rep

A cada tarefa concluída, o servidor credita 5 pontos na skill `cityworker`:

```lua
exports["cw-rep"]:updateSkill(src, "cityworker", 5)
```

Como a chamada não é condicional, o `cw-rep` precisa estar rodando.

### Script de combustível

Com `FuelScript.enable = true`, o veículo recebe combustível cheio via `exports[Config.FuelScript.script]:SetFuel(vehicle, 100.0)`. Com `false`, o recurso usa o statebag `Entity(vehicle).state.fuel = 100`, compatível com `ox_fuel` e derivados.

### Chaves do veículo (QB)

No bridge QB, `handleVehicleKeys` dispara `vehiclekeys:client:SetOwner` com a placa gerada. No bridge ESX a função é um stub vazio — se o seu servidor ESX usa sistema de chaves, implemente ali.

---

## Estrutura de arquivos

```
randol_cityworker/
├── bridge/
│   ├── client/
│   │   ├── esx.lua           — notificações, estado de login e chaves (stub) no ESX
│   │   └── qb.lua            — notificações, estado de login e vehiclekeys no QB
│   └── server/
│       ├── esx.lua           — GetPlayer, AddMoney (converte cash→money), handleExploit (kick)
│       └── qb.lua            — GetPlayer, AddMoney, handleExploit (ban na tabela bans)
├── cl_cityworker.lua         — blip, NPC, target, zona de reparo, progressbar
├── sv_cityworker.lua         — spawn do veículo, sorteio de tarefa, pagamento, limpeza
├── config.lua                — config compartilhada (NPC, coords, combustível)
├── sv_config.lua             — config do servidor (pagamento, locais, veículo)
└── fxmanifest.lua
```
