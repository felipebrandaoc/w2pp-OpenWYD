# OpenWYD Architecture Baseline

## Purpose

Este documento registra a arquitetura observada da implementação Go antes das
customizações próprias da fork, na revisão
`98286fdf01202f503523e89d3e50b2183f00c36c`.

O baseline deriva exclusivamente da inspeção estática realizada no Milestone 4.
Não representa validação em execução com WYD.exe, execução de testes ou auditoria
exaustiva de paridade com o legado. Os caminhos são relativos à raiz do projeto.

Os termos de estado usados neste documento têm significados distintos:

- **implemented**: existe um caminho concreto na implementação inspecionada;
  não significa cobertura completa nem validação ponta a ponta.
- **partial**: há implementação, mas também lacunas ou adaptações identificadas.
- **placeholder**: o código usa explicitamente um valor, formato ou comportamento
  substituto enquanto a implementação definitiva não está disponível.
- **unverified**: a correspondência com o legado ou com o cliente não foi
  confirmada; isso não implica necessariamente ausência de implementação.

## Core architecture

| Componente | Responsabilidade observada | Referências principais |
| --- | --- | --- |
| `tmserver` | Conexão CPSock com o cliente, sessões, estado runtime, simulação e gameplay. | `tmserver/cmd/tmserver/main.go`, `tmserver/internal/world/world.go`, `tmserver/internal/handler/dispatch.go` |
| `dbserver` | Autenticação e persistência expostas por gRPC; conversão entre protobuf e domínio; acesso ao PostgreSQL. | `dbserver/cmd/dbserver/main.go`, `dbserver/internal/grpcsrv/server.go`, `dbserver/internal/grpcsrv/mapping.go` |
| `binserver` | Serviço opcional consultado pelo gate de billing na entrada do personagem. A inspeção confirmou a integração no TM, sem estabelecer cobertura completa das regras internas do serviço. | `binserver/cmd/binserver/main.go`, `tmserver/internal/binclient/client.go`, `tmserver/internal/world/billing.go`, `tmserver/internal/handler/character.go` |
| `webserver` | API web de contas e administração; possui integração com conteúdo para catálogo/edição de NPCs e itens. Não participa do caminho CPSock de gameplay. | `webserver/cmd/webserver/main.go`, `internal/store/npc.go` |
| PostgreSQL | Armazena contas, personagens, itens, affects e configurações/estados persistentes de sistemas auxiliares. | `internal/store/store_live.go`, `internal/domain/domain.go`, `internal/migrations/` |
| `Release/` | Dados legados consumidos pelos loaders Go quando o diretório de conteúdo é configurado. | `tmserver/internal/content/`, `internal/npctemplate/npctemplate.go` |
| `Source/` | Referência C++ de comportamento, protocolo e formatos; não é código executado pelo servidor Go no fluxo inspecionado. | `Source/Code/Basedef.h`, `Source/Code/Basedef.cpp`, `Source/Code/CPSock.cpp`, `Source/Code/TMSrv/`, `Source/Code/DBSrv/` |

O TM usa `world.Persistence` como interface e `dbclient.Client` como adaptador
gRPC. Sem `-dbserver`, seleciona `NopPersistence`, que não autentica contas reais.
Sem billing configurado, utiliza `AllowAllBilling`.

O contrato de persistência está em `api/db/v1/db.proto`; o acesso SQL usa pgx v5.
As credenciais de transporte interno passam por `internal/secure/tls.go`.
Não foi verificada a configuração efetiva de processos em execução.

## World ownership model

`World`, em `tmserver/internal/world/world.go`, concentra o estado mutável do
jogo. Durante a execução, `World.Run` é seu proprietário único: handlers de
pacotes, ticks e callbacks modificam esse estado serialmente na goroutine do loop.
O bootstrap popula o mundo sequencialmente antes de iniciar o loop.

`World.Run` processa eventos e callbacks, dando prioridade aos callbacks já
disponíveis. `World.Go`, em `tmserver/internal/world/api.go`, executa trabalho
bloqueante fora do loop e devolve uma função para aplicação dentro dele.
O callback vinculado à sessão verifica se a sessão original ainda ocupa o slot.
`World.GoDetached` permite callbacks não vinculados a uma sessão específica.

| Estrutura | Papel |
| --- | --- |
| `events chan event` | Conexão, frames, desconexão e ticks; capacidade padrão 1024. |
| `callbacks chan event` | Resultados assíncronos; capacidade 256. |
| `Session.out` | Fila de saída por conexão; capacidade padrão 64. |
| `done` | Sinaliza encerramento para as goroutines de I/O. |
| `saveWG` | Rastreia saves assíncronos específicos para espera no shutdown. |
| `respawnQueue` | Guarda mobs aguardando reaparecimento. |

`Session` contém estado da conexão, conta, personagem selecionado e controles de
protocolo. `Entity` representa tanto players quanto mobs/NPCs. Ambas estão em
`tmserver/internal/world/session.go`.

- `sessions`: 1000 posições; slot zero reservado.
- `entities`: 25000 posições; players abaixo de 1000 e mobs/NPCs a partir de 1000.
- `ground`: 5000 posições para objetos de chão; slot zero reservado.
- `Grid`: índice espacial, dimensão padrão 4096, em `world/grid.go`.
- Cargo por conta, geradores, guildas e estados de eventos também vivem em memória.

`world/event.go` define os eventos e os loops de leitura/escrita de sockets.
`world/tick.go` emite `tickEvent` a cada segundo por padrão; a simulação ocorre
em `Dispatcher.Tick`, dentro do loop. Essa cadência é **unverified** em relação
ao timing original.

Não há autorização para mutações concorrentes do estado do mundo nem para
introduzir locks como substituto desse modelo. Goroutines de rede trocam mensagens
com o loop. Snapshots para persistência são capturados antes do I/O assíncrono.
No encerramento, o código faz saves síncronos deliberados após interromper a
operação normal e aguarda os saves assíncronos rastreados.

## Runtime flows

### Login

**Estado: implemented no caminho principal; alguns detalhes de protocolo são unverified.**

O listener do TM usa `:8281` por padrão. `World.Serve` aceita a conexão;
`handleConn` distingue CPSock de uma consulta HTTP de status.
`connectEvent.apply` cria `Session` em `UserAccept` e `Entity` em `MobUserDock`.
A sessão de transporte existe antes da autenticação.

`readLoop` faz framing e decode, emitindo `frameEvent`. O dispatcher encaminha
`MsgAccountLogin` (`0x020D`) para `accountLogin`, que valida tamanho, versão,
estado e limite de falhas. `World.Go` encaminha autenticação ao dbserver.
`Store.AccountByName` consulta `account`; `secret.VerifySecret` verifica Argon2id.
O adaptador também busca seleção de personagens, cargo e entregas pendentes.
O callback associa a conta, instala cargo e passa para `UserSelChar`.

Arquivos principais:

- `tmserver/cmd/tmserver/main.go`
- `tmserver/internal/world/server.go`, `edge.go`, `event.go`, `session.go`
- `tmserver/internal/protocol/framing.go`, `codec.go`
- `tmserver/internal/handler/dispatch.go`, `login.go`
- `tmserver/internal/dbclient/client.go`
- `dbserver/internal/grpcsrv/server.go`
- `internal/store/store_live.go`, `internal/secret/secret.go`

### Character load

**Estado: implemented, com adaptações e valores iniciais descritos como placeholder.**

A lista da conta vem de `ListCharacters`. `MsgCharacterLogin` (`0x0213`) aciona
`characterLogin`: valida slot/estado, passa para `UserCharWait`, consulta billing
e executa `LoadCharacter` fora do loop.

O fluxo de conversão é PostgreSQL → `domain.Character` → protobuf →
`world.CharacterState` → `Entity`/`Session`/`Grid`.
`completeCharacterLogin` remove itens expirados, prepara equipamento inicial
quando necessário, restaura affects e recalcula stats com `deriveBaseScore` e
`refreshScore`. Depois passa para `UserPlay`, envia a confirmação e chama
`enterWorldView`.

`SaveX/SaveY` são o ponto salvo da Gema Estelar. A posição livre de entrada não
vem no contrato atual; cargas reais normalmente usam o spawn de `LastCity`.

Arquivos principais:

- `tmserver/internal/handler/character.go`, `item.go`, `score_derive.go`
- `tmserver/internal/world/persistence.go`, `api.go`
- `tmserver/internal/dbclient/client.go`
- `dbserver/internal/grpcsrv/server.go`, `mapping.go`
- `internal/store/store_live.go`
- `tmserver/internal/content/basemob.go`

### Movement

**Estado: partial.**

`MsgAction`, `MsgAction2` e `MsgAction3` chegam a `action`. O handler verifica
estado, vida, trade, pacote, regras de Illusion, janela temporal, limites do mapa
e restrições de Castle. `SetEntityPos` atualiza a posição autoritativa para o
destino e o grid; a rota recebida é armazenada.

`moveMulticast` reconcilia as áreas antiga/nova de visão e envia movimento,
criação e remoção de entidades. Não há simulação contínua do trajeto do player
nesse handler. A janela temporal não constitui validação completa de velocidade
ou colisão.

Arquivos principais:

- `tmserver/internal/handler/movement.go`, `view.go`
- `tmserver/internal/world/api.go`, `grid.go`
- `tmserver/internal/protocol/messages.go`

### Combat

**Estado: partial, com caminho autoritativo de dano implemented.**

`MsgAttack`, `MsgAttackOne` e `MsgAttackTwo` chegam a `attack`. O servidor valida
estado, cadência, alvos, regras PvP e condições de skills, calculando o dano sem
aceitar o valor alegado pelo cliente como autoritativo.

`HitInput`, `Damage`, `SkillDamage`, `ResolveHit` e `DoubleCritical` concentram
fórmulas. O dano físico parte de ataque menos metade da defesa e recebe variação,
mastery, críticos e parry. Skills possuem caminhos adicionais de custo e efeitos.
O resultado atualiza `Entity.HP`, limitado a zero, e é enviado aos envolvidos.

Morte de mob passa por `mobKilled` e `DespawnMob`; morte PvP por `pvpKilled`.
Ataques de mobs aplicam a perda de EXP por morte em seu próprio fluxo.
`restart` e handlers de skills implementam caminhos de retorno/ressurreição.

O RNG de gameplay é `World.Rand()`, um `rng.MSVC` com seed inicial 1:
`state = state*214013 + 2531011`, resultado `(state >> 16) & 0x7FFF`.
`Intn(n)` usa módulo, preservando o comportamento do CRT; a ordem das chamadas
é relevante para paridade.

Arquivos principais:

- `tmserver/internal/handler/combat.go`, `hpmp.go`, `mobai.go`, `mobkilled.go`
- `tmserver/internal/handler/pvpkilled.go`, `death_exp.go`, `character.go`
- `tmserver/internal/combat/combat.go`, `critical.go`, `skill.go`
- `tmserver/internal/rng/rng.go`

### Experience / level

**Estado: partial; curva Mortal/Arch implemented e curva Celestial placeholder.**

`grantExp` chama `SoloExpReward` com EXP do template, níveis, tier e bônus.
O cálculo aplica `ExpApply`, escala por nível, gate de recompensa, divisores de
tier, fator 0,6, teto, bônus e eventos. `applyMobExp` altera EXP;
`applyLevelUps` percorre thresholds, respeita locks, concede pontos, recalcula
stats e restaura HP/MP. `grantDirectExp` atende recompensas fixas.

`nextLevel` é a tabela compilada Mortal/Arch; limite interno 399.
`nextLevel2` usa `celestialPlaceholderCurve`, uma rampa sintética que soma
`1.000.000 + i*100.000` por etapa; limite interno Celestial 199.
Essa curva não deve ser descrita como paridade comprovada com o cliente.

EXP base vem dos templates de mobs. Flags Double/Newbie/Kefra e configuração
de eventos influenciam a recompensa. Equipamentos, fadas e affects fornecem
bônus. Water/Nightmare possuem distribuição específica; party genérica não está
implementada. Thresholds gerais não são carregados de um arquivo externo.
`MobExpForLevel` serve à ferramenta que regrava EXP de templates, não substitui
o cálculo runtime de recompensa.

Arquivos principais:

- `tmserver/internal/level/level.go`, `nextlevel.go`, `expreward.go`, `mobexp.go`
- `tmserver/internal/level/water_exp.go`, `nightmare.go`
- `tmserver/internal/handler/mobkilled.go`, `exp_bonus.go`, `worldevent.go`
- `tmserver/internal/handler/water.go`, `nightmare_exp.go`
- `tmserver/internal/exptool/exptool.go`

### Drops

**Estado: partial; comparação de sorteio marcada como unverified.**

O `Carry` do mob é sua lista de loot, proveniente do template.
`EffectiveDropRate` aplica chances compiladas por slot e ajustes por nível;
`Drops` usa o RNG do mundo. O bônus individual é atualmente passado como zero.

`putMobDrop` entrega loot comum diretamente ao inventário do recebedor.
Inventário cheio perde a recompensa, sem fallback para o chão. Ouro também é
creditado diretamente. O evento global tem caminho próprio de drop no inventário.

O chão tem fluxo distinto: `dropItem` cria `GroundItem` e limpa a origem;
`getItem` valida ID, existência, distância e slot, transfere o item e remove o
objeto do chão. IDs de chão usam offset 10000 na rede.
O broadcast de criação no chão permanece adiado no código inspecionado.

Arquivos principais:

- `tmserver/internal/handler/mobkilled.go`, `worldevent.go`, `item.go`
- `tmserver/internal/loot/loot.go`
- `tmserver/internal/world/items.go`

### Mobs

**Estado: partial, com spawn, combate, patrulha e respawn implemented.**

`NPCGenerator` descreve conteúdo; `Generator` mantém a receita e população viva.
O bootstrap lê geradores e templates, normaliza formatos legados para 816 bytes,
aplica overrides opcionais e chama `GenerateMob`/`SpawnMobAt`.
`Entity` guarda stats, alvo, grupo, rota, origem e template para reaparecimento.

`Dispatcher.Tick` coordena aquisição de inimigos, combate, perseguição,
patrulha, retirada e summons. `route.Next` usa os mapas quando disponíveis.
Geradores com `MinuteGenerate > 0` repõem população pelo ciclo próprio; outros
mobs elegíveis usam fila com atraso padrão de 15 segundos. Summons e eventos
especiais possuem exclusões. Spawn imediato no boot e essa fila são adaptações
deliberadas. Uma tentativa de respawn sem espaço é descartada.

Arquivos principais:

- `tmserver/internal/content/npc.go`
- `internal/npctemplate/npctemplate.go`
- `tmserver/internal/world/session.go`, `generator.go`, `api.go`, `respawn.go`
- `tmserver/internal/handler/mobai.go`, `summon.go`
- `tmserver/internal/route/route.go`, `tmserver/internal/mobstat/mobstat.go`
- `tmserver/cmd/tmserver/main.go`

### NPCs

**Estado: partial.**

NPCs compartilham `Entity`; `Merchant`, `Grade`, `Clan` e `NonCombatNPC`
orientam serviços e participação em combate. Há conteúdo de arquivos e overlay
opcional do banco por `-npc-editing`, com polling de alterações.
O overlay de stats por `-mob-stat-editing` é aplicado no boot.

Lojas usam `Carry` do NPC como estoque e preços do catálogo/overrides.
`reqShopList`, `buy` e `sell` implementam operações; imposto é zero e venda está
limitada ao Carry. `quest` despacha serviços por classificação do NPC.
Há caminhos para Quest 256, Perzen, Mestre Grifo, capas/reinos, Terra Mística,
Black Oracle, Sephirot, Kibita e criação Arch pelo rei. Tipos restantes podem
chegar ao log `quest NPC not implemented`.

Arquivos principais:

- `tmserver/internal/content/npc.go`, `tmserver/internal/world/api.go`
- `tmserver/internal/handler/npcconfig.go`, `shop.go`, `misc.go`, `cargo.go`
- `tmserver/internal/npccfg/npccfg.go`, `tmserver/internal/dbclient/npcconfig.go`
- `dbserver/internal/grpcsrv/npcconfig.go`, `internal/store/npc.go`

### Items

**Estado: partial, com inventário e equipamento implemented.**

`world.Item` contém índice, efeitos e expiração; `GroundItem` representa chão.
`Entity` contém Carry de 64 slots e Equip de 16; cargo possui 128 por conta.
`content.ItemList` fornece efeitos, preços, requisitos, posições, grades e alcance.
`protocol.SelItem` representa o item no protocolo e `domain.Item` na persistência.

Handlers cobrem uso, equipamento, movimentação, divisão, destruição, cargo,
troca, autotrade, refino e composição. Acessibilidade dos slots é validada.
Equipamento/affects recompõem stats a partir do estado base para evitar acumulação
de bônus. Slots não vazios são persistidos em `item`, classificados como
`char_equip`, `char_carry` ou `account_cargo`.

Arquivos principais:

- `tmserver/internal/world/items.go`, `session.go`, `persistence.go`
- `tmserver/internal/content/catalog.go`
- `tmserver/internal/handler/item.go`, `carry.go`, `cargo.go`, `trade.go`, `autotrade.go`
- `tmserver/internal/handler/refine.go`, `combine.go`, `combine_variants.go`, `score_derive.go`
- `tmserver/internal/refine/`, `tmserver/internal/combine/`
- `tmserver/internal/protocol/carry.go`, `mob.go`, `shop.go`
- `internal/domain/domain.go`, `internal/store/store_live.go`

### Persistence

**Estado: implemented para os campos controlados pelo runtime; atualização partial por contrato.**

Saves ocorrem na desconexão, logout para seleção, shutdown e ações específicas
de guilda/progressão/quests/composição. `SaveCharacterThen` aguarda o resultado
antes de continuar a transição; `SaveCharacterAsync` captura snapshot e grava
fora do loop. Não foi encontrado autosave periódico geral ligado ao tick.

O save inclui nível, EXP, ouro, atributos, HP/MP, Carry, equipamentos, affects,
habilidades/barras, especializações, guilda/clã, cidade, ponto salvo, tier,
flags de progressão, karma e entradas de eventos. Campos importados fora da
autoridade desse fluxo permanecem intactos. A posição livre atual não equivale
ao ponto salvo persistido.

Arquivos principais:

- `tmserver/internal/world/world.go`, `event.go`, `persistence.go`
- `tmserver/internal/handler/character.go`
- `tmserver/internal/dbclient/client.go`, `api/db/v1/db.proto`
- `dbserver/internal/grpcsrv/server.go`, `mapping.go`
- `internal/store/store_live.go`, `store.go`, `internal/domain/domain.go`

## Persistence flow

```text
Entity / Session
→ World.CharacterSaveFor
→ dbclient
→ gRPC SaveCharacter
→ dbserver
→ Store.SaveCharacter
→ PostgreSQL
```

`CharacterSaveFor` captura o estado autoritativo em `CharacterSave`.
`dbclient.characterSaveToProto` prepara o contrato; o servidor gRPC converte
para domínio e chama o store. `Store.SaveCharacter` atualiza os campos selecionados
e substitui equipamento, Carry e affects em uma transação.

Cargo usa transação separada. A atomicidade do save do personagem não deve ser
interpretada como atomicidade entre todos os participantes de trade/cargo.
Existem RPCs específicos para operações persistentes, como compra de capa de reino.
Algumas falhas de save apenas geram logs; esta inspeção não estabeleceu garantias
completas de recuperação após falha ou encerramento abrupto.

Migrations relevantes em `internal/migrations/`:

| Arquivos | Área |
| --- | --- |
| `0001_init.up.sql` | `account`, `character`, `item`, `affect` |
| `0002_last_city.up.sql`, `0003_item_expires.up.sql`, `0004_special.up.sql` | Cidade, expiração, especializações |
| `0005_npc_editing.up.sql`, `0007_npc_shop_item_quantity.up.sql`, `0018_npc_generator_catalog.up.sql` | NPCs, lojas, preços, geradores |
| `0013_mob_template_stats.up.sql` | Stats/equipamento de templates |
| `0008_mobextra_skill_soul.up.sql`, `0012_celestial_quest.up.sql`, `0013_terra_mistica_quest.up.sql`, `0019_arch_quest_flags.up.sql`, `0020_arch_celestial_progression.up.sql`, `0022_arch_crystal_stage.up.sql` | Skills e progressão |
| `0012_character_fame.up.sql`, `0016_pontos_caos.up.sql` | Fame e karma |
| `0012_guild_system.up.sql`, `0012_duel_ranking.up.sql` | Guildas, zonas, Castle e PvP |
| `0015_world_event_config.up.sql` | Configuração de eventos |
| `0022_kefra_ticket.up.sql`, `0023_kefra_state.up.sql`, `0024_nightmare_entries.up.sql` | Kefra e Nightmare |
| `0008_donate_shop.up.sql` | Loja donate e `delivery_queue` |

`internal/migrations/migrations.go` embute os SQLs;
`internal/store/migrate.go` executa migrations. O bootstrap `runServe` do dbserver
chama `store.Migrate` automaticamente. Nenhum desses fluxos foi executado para
produzir este baseline.

## Legacy content

`Source/` é referência de comportamento e formato, não código executado pelo
servidor Go no fluxo inspecionado. Entre as referências estão:

- `Source/Code/Basedef.h` e `Basedef.cpp`: estruturas, constantes e fórmulas.
- `Source/Code/CPSock.h` e `CPSock.cpp`: framing e obfuscação.
- `Source/Code/TMSrv/CUser.cpp`, `CMob.cpp`, `Server.cpp`, `ProcessSecMinTimer.cpp`:
  sessão e simulação.
- `Source/Code/TMSrv/_MSG_AccountLogin.cpp`, `_MSG_CharacterLogin.cpp`,
  `_MSG_Action.cpp`, `_MSG_Attack.cpp`, `MobKilled.cpp`: fluxos de jogo.
- `Source/Code/TMSrv/CItem.cpp`, `_MSG_GetItem.cpp`, `_MSG_DropItem.cpp`,
  `_MSG_Quest.cpp`, `_MSG_REQShopList.cpp`, `SendFunc.cpp`, `GetFunc.cpp`:
  itens, NPCs e operações auxiliares.
- `Source/Code/DBSrv/CFileDB.cpp`, `Server.cpp`: persistência legada em arquivos.
- `Source/Code/ExpTool/main.cpp`: referência da ferramenta de EXP.

Dados relevantes carregados de `Release/`, quando configurado `-content` ou
`W2PP_CONTENT`:

| Caminho | Uso |
| --- | --- |
| `Release/Common/ItemList.csv` | Catálogo de itens |
| `Release/Common/SkillData.csv` | Skills |
| `Release/Common/Settings/CompRate.txt` | Composição |
| `Release/Common/Settings/SancRate.txt` | Refinamento |
| `Release/Common/Settings/QuestsRate.txt` | Taxas de regras específicas |
| `Release/Common/Settings/CastleQuest.txt` | Castle/Zakum |
| `Release/Common/serv00.htm` | Status HTTP do canal |
| `Release/DBsrv/run/BaseMob/{TK,FM,BM,HT}` | Templates iniciais por classe |
| `Release/TMsrv/run/NPCGener.txt` | Geradores, grupos, rotas e falas |
| `Release/TMsrv/run/npc/*` | Templates de mobs/NPCs, incluindo Vine |
| `Release/TMsrv/run/BaseSummon/*` | Templates de summons |
| `Release/TMsrv/run/InitItem.csv` | Objetos estáticos |
| `Release/TMsrv/run/HeightMap.dat`, `AttributeMap.dat` | Navegação |

Loaders principais: `tmserver/cmd/tmserver/main.go`,
`tmserver/internal/content/catalog.go`, `skilldata.go`, `rates.go`, `npc.go`,
`basemob.go`, `basesummon.go`, `inititem.go`, `gamemap.go`, `castlequest.go`, e
`internal/npctemplate/npctemplate.go`.

O dbserver pode reconciliar o catálogo de NPCs a partir desse conteúdo; o
webserver também integra conteúdo para catálogo/edição. Os binários legados de
`Release/` não fazem parte da execução das regras Go identificadas.

## Known incomplete or placeholder areas

| Área | Estado | Evidência / limite observado |
| --- | --- | --- |
| Movement validation | partial | `handler/movement.go`: clamp completo de velocidade, correção de saltos e redirecionamento de destino ocupado adiados. |
| Change city | placeholder | `handler/movement.go`: `villageAt` retorna `-1`. |
| Celestial EXP | placeholder | `level/nextlevel.go`: `celestialPlaceholderCurve` gera thresholds sintéticos. |
| Generic party EXP | partial | `handler/mobkilled.go`, `level/expreward.go`: distribuição genérica pendente; Water/Nightmare têm caminhos próprios. |
| Recompensas por nível | partial | `handler/mobkilled.go`: `DoItemLevel` indicado como adiado. |
| Individual drop bonus | placeholder | `handler/mobkilled.go`: passa zero a `EffectiveDropRate`. |
| Comparação do drop | unverified | `loot/loot.go`: comparação final descrita como não confirmada. |
| Ground-item broadcast | partial | `handler/item.go`: broadcast de criação no chão explicitamente adiado. |
| Confirmações de drop/pickup | placeholder / unverified | `handler/item.go`: `slotPayload` substitui layouts ainda não confirmados. |
| NPC/service handlers | partial | `handler/misc.go`: fallback para NPC não implementado; cash/cápsula e `PutoutSeal` pendentes. |
| Lojas | partial | `handler/shop.go`: imposto zero e venda limitada ao Carry. |
| PvP legacy rules | partial / unverified | `handler/pvpkilled.go`: lacunas em geografia, regras de penalidade e mecanismos legados de perda de EXP. |
| Composition gaps | partial / placeholder | `handler/combine_catalog.go`, `combine_variants.go`, `combine.go`: famílias/regras incompletas e caminhos substitutos. |
| Cadência de AI/respawn | unverified | `world/tick.go`, `world/respawn.go`: defaults e adaptações, não paridade temporal comprovada. |
| Autosave periódico geral | partial | Não encontrado nos caminhos de tick/save inspecionados; saves existem em transições e ações específicas. |
| Autotrade EF_NOTRADE | partial | `handler/autotrade.go:111`: `TODO(#115)` para rejeitar esses itens. |
| Mensagens auxiliares | placeholder / unverified | `handler/notice.go`, `protocol/cargo.go`, `handler/trade.go`: formatos ainda não confirmados. |

Os caminhos abreviados da tabela pertencem a `tmserver/internal/`.
A busca literal realizada em `tmserver`, `dbserver`, `internal` e `api` encontrou
o TODO de autotrade acima e nenhum `FIXME`. A ausência desses marcadores não
significa ausência de lacunas: muitos comentários usam `UNVERIFIED`, `deferred`
ou `placeholder`.

Comentários desatualizados não devem ser usados isoladamente como diagnóstico:

- `level/level.go` diz que tiers não são modelados, mas há lógica por tier e locks.
- `handler/mobai.go` menciona pathfinding/patrulha/summons pendentes, mas existem
  `route.Next`, `mobRoam` e `summonTick`.
- `level/expreward.go` menciona fada pendente, mas `handler/exp_bonus.go` a inclui.
- `handler/character.go` menciona save futuro, mas chama `SaveCharacterThen`.
- `world/world.go` descreve guildas apenas em memória, mas há RPCs e tabelas.

Documentos antigos de `docs/agents/` e partes de `docs/migration/` descrevem o
C++ e seus arquivos de conta/mensagens DB. Não devem ser confundidos com a
persistência Go atual por gRPC, Argon2id e PostgreSQL.

## Architectural rules for future development

As regras abaixo orientam as próximas mudanças da fork; não são todas mecanismos
automaticamente impostos pelo código atual.

1. Não mutar estado de World fora de `World.Run` durante a execução. Manter o
   bootstrap sequencial anterior ao loop; não introduzir concorrência no estado.
2. Usar `World.Go` para I/O bloqueante iniciado por handlers, capturando entradas
   no loop e aplicando resultados por callback. Usar `GoDetached` quando o
   resultado não pertencer a uma sessão; nunca acessar entidades vivas na goroutine.
3. Não alterar protocolo ou layout binário sem necessidade demonstrada. Consultar
   a documentação de migração, offsets explícitos e comportamento do cliente.
4. Preservar a ordem das chamadas ao RNG quando relevante para paridade. Evitar
   sorteios incidentais no fluxo compartilhado de gameplay.
5. Criar novas migrations para evolução do schema, em vez de reescrever migrations
   existentes. Considerar dados persistidos e compatibilidade de upgrades.
6. Manter `Source/` como referência, evitando alterações incidentais no legado.
7. Manter mudanças pequenas e delimitadas para facilitar revisão e sincronização
   futura com upstream.
8. Separar correções de comentários, implementação de lacunas e customizações de
   gameplay. Não promover um placeholder a comportamento definitivo sem decisão
   explícita e validação apropriada.

## Full flow

```text
WYD.exe → tmserver → World.Run → gameplay → dbclient → dbserver → PostgreSQL
             │           │          │          │          │
           CPSock     events /   Entity /     gRPC      Store / pgx
                      callbacks  Session /             transações
                                 Grid

PostgreSQL → dbserver → dbclient → callback → World.Run
World.Run → Session.out → writeLoop → CPSock → WYD.exe

Release/ → loaders → catálogos / templates / mapas → world / gameplay
binserver → gate opcional de entrada do personagem
Source/ → referência de comportamento e formato
```

Movimento e combate operam sobre o estado em memória; não implicam um RPC ao
banco para cada ação. A persistência ocorre nos pontos de save identificados.
