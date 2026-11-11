# Pitfall: agente lógico em Prolog

O Pitfall Harry entra num labirinto 12 x 12 pela casa **[1,1]** sem conhecer o mapa. Ele só sente
o que está perto dele:

| Percepção | Onde é sentida | Indica |
| --- | --- | --- |
| brisa | nas 4 vizinhas de um poço | poço (morte, -1000) |
| passos | nas 4 vizinhas de um inimigo | inimigo (d: -25 a -50 de energia, D: -50 a -80) |
| palmas | nas 4 vizinhas de um teletransporte | teletransporte (vai para uma casa sorteada) |
| brilho | na própria casa | ouro (+500) |
| reflexo | na própria casa | item de energia (+50, máximo 100) |

O objetivo é **pegar os 3 ouros e voltar para [1,1] vivo** com a maior pontuação possível. Andar e virar
custam 1 ponto e pegar custa 5.

A inteligência fica toda em Prolog (`agente.pl`). O Python só desenha e chama o Prolog.

| Parte | Técnica | Aula |
| --- | --- | --- |
| Base de conhecimento | Fatos dinâmicos (`memory`, `certeza`, `visitado`, `percepcao`) atualizados a cada percepção | 07.1, 08, 09 |
| Inferência | Casa OK e regra de diagnóstico "sentiu brisa e só uma vizinha pode ter poço, então o poço está nela" | 07.1, 07.2 |
| Escolha da ação | Sistema de ação-valor: ótimas, boas, arriscadas, voltar e último recurso, nessa ordem | 07.2 |
| Planejamento | Busca em largura só por casas seguras | 03 |
| Risco | Chance aproximada de perigo em cada casa incerta | 02 (agente baseado em utilidade) |

## Resultados

Gerados com `python rodar_testes.py --aleatorios 200`. O teletransporte e o dano dos inimigos são
sorteados, por isso cada mapa fornecido rodou 30 vezes, com sementes de 0 a 29.

| Mapa | Partidas | Pontuação | Ouro | Sobreviveu | Saiu com 3 ouros | Ações |
| --- | --- | --- | --- | --- | --- | --- |
| `mapa_facil.pl` | 30 | **1234** | 3 | 30/30 | 30/30 | 255 |
| `mapa_medio.pl` | 30 | **1130** | 3 | 30/30 | 30/30 | 359 |
| `mapa_dificil.pl` | 30 | **1151** | 3 | 30/30 | 30/30 | 338 |
| 200 mapas aleatórios | 200 | 937,0 (média) | 2,33 (média) | 190/200 | 148/200 | 179,5 (média) |

Nos mapas fornecidos o agente nunca entra num teletransporte nem leva dano, então todas as 30 partidas
de cada mapa saem iguais. Os **mapas aleatórios** têm os mesmos elementos do enunciado (2 d, 2 D, 8 poços,
3 ouros, 4 teletransportes e 3 itens de energia) em casas sorteadas, com [1,1] livre. Eles mostram como o
agente se sai em situações que não vimos ao escrever as regras.

Nos 200 aleatórios, das 10 mortes, 8 foram em poço e 2 por inimigo. Quase todas vieram de riscos que
o agente aceitou: cair de um teletransporte direto num poço, ou uma casa com chance de poço < 20% que
tinha o poço. Em alguns mapas sorteados nem é possível pegar os 3 ouros sem arriscar muito; nesses
casos o agente volta e sai com o que tem.

Cada partida está em `resultados/partidas.csv`, com mapa, semente, pontuação, ouro, energia, ações e
resultado.

## Como o agente pensa

A cada jogada o Python chama `executa_acao(X)`. O agente faz três coisas, como o KB-Agent da Aula 07.1:

1. **Atualiza o que sabe.** O motor já guardou o que ele sentiu (`atualiza_obs`). O agente roda
   `deduz_perigos` para tirar conclusões novas.
2. **Escolhe a ação.** Testa as regras de `acao/1` em ordem de prioridade; vale a primeira que der certo.
3. **Devolve só a primeira ação do plano.** Na próxima jogada ele replaneja com o que aprendeu.

### Base de conhecimento

| Fato | Significado | Quem escreve |
| --- | --- | --- |
| `visitado(X,Y)` | O agente já esteve em [X,Y] e saiu | motor (`andar`) |
| `memory(X,Y,L)` | O que **pode** ter em [X,Y]; numa casa visitada, o que tem de fato | motor (`atualiza_obs`, `set_real`) e agente (`confirma`) |
| `certeza(X,Y)` | A memória de [X,Y] é certa | motor e agente |
| `percepcao(X,Y,L)` | O que o agente sentiu quando estava em [X,Y] | motor (`guarda_percepcao`) |
| `posicao/3`, `energia/1`, `ouros/1` | Estado do próprio agente | motor |

Uma casa ainda não visitada começa com a lista do que foi sentido ao lado dela, por exemplo `[brisa]`. A cada
nova vizinha visitada, o motor faz a interseção: se uma vizinha **não** sentiu brisa, aquela casa não tem poço.

### Inferência

| Regra | Em Prolog | De onde vem |
| --- | --- | --- |
| Casa OK: tem certeza e nenhum perigo | `seguro(X,Y)` | "OK" da Aula 07.2 |
| Casa sem nenhuma observação ao lado é OK | `observacao_vazia` (código-base) | ¬Brisa ⇒ ¬Poço nas vizinhas |
| Se [V] sentiu Z e só **uma** vizinha de [V] pode ter Z, então Z está nela | `deduz_perigos` | `Arejado(s) ⇒ ∃r Adjacente(r,s) ∧ Poço(r)` (Aula 07.1) |
| Cada casa tem uma coisa só: confirmado Z, a memória vira `[Z]` | `confirma/3` | regra do mapa |

O código-base tinha uma versão da terceira regra (`observacao_certeza`), mas ela tinha dois problemas:

- só olhava a posição atual;
- contava só as vizinhas com memória exatamente `[brisa]`, ignorando as com `[brisa, passos]`. Assim
  podia marcar um poço falso, a brisa ficava "explicada" e o agente arriscava o poço verdadeiro.

Por isso ela saiu de `atualiza_obs`. A `deduz_perigos` faz a dedução certa em todas as casas visitadas,
e é por isso que o agente guarda a percepção de cada uma.

### Escolha da ação (sistema de ação-valor)

As regras de `acao/1` estão nesta ordem:

| # | Tipo | Regra | Por quê |
| --- | --- | --- | --- |
| 1 | Ótima | Sentiu brilho: `pegar` | +495 líquido |
| 2 | Ótima | Sentiu reflexo e energia < 100: `pegar` | Com 100 o item não serve; ele lembra onde está |
| 3 | Fuga | Está na casa de um inimigo: anda para frente se a casa da frente for aceitável; senão, vai para a vizinha segura mais próxima | O motor tira energia a cada ação nessa casa, até ao virar |
| 4 | Energia | Energia ≤ 50: volta para um item de energia já visto | Com pouca energia, um inimigo pode matar |
| 5 | Boa | Falta ouro: vai para a casa OK não visitada mais próxima | Explorar sem risco |
| 6 | Arriscada | Falta ouro e não há casa OK nova: vai para a casa aceitável de menor risco | Ver "Avaliação de risco" |
| 7 | Objetivo | Em [1,1]: `sair`. Senão: planeja a volta para [1,1] | Pegou tudo ou não vale mais arriscar |
| 8 | Objetivo | Sem caminho seguro e energia > 80: volta passando por um inimigo conhecido | O dano máximo é 80 |
| 9 | Último recurso | Sem caminho seguro para casa: casa incerta ou teletransporte conhecido de menor risco | Ficar parado não termina o jogo |
| 10 | Chute | Ação aleatória | Sugestão do professor; decidiu 1 vez em cerca de 37 mil jogadas nos testes |

### Planejamento

`mover_para(Alvos, Acao)` roda uma **busca em largura** (Aula 03) da posição do agente até o alvo mais
próximo, passando só por casas seguras. O alvo pode estar incerto, como numa jogada arriscada.

- **Por que BFS e não A\*:** explorando, o destino é "qualquer casa OK nova", não um ponto fixo para a
  heurística. Como todo passo custa 1, a BFS acha o menor caminho (tabela de comparação da Aula 03).
- **Estados repetidos:** a lista `Vistas` impede colocar a mesma casa duas vezes na fila.
- **Menos giros:** a BFS expande primeiro a casa à frente. Entre caminhos do mesmo tamanho, prefere o que
  segue reto.
- **Replanejamento:** só a primeira ação do caminho é usada, e na jogada seguinte o plano é refeito. É o
  "monitoramento da execução com replanejamento" da Aula 11.

### Avaliação de risco

Quando acabam as casas OK e ainda falta ouro, o agente estima a **chance** de cada perigo possível numa
casa incerta. A chance é a maior entre:

- **a priori:** perigos daquele tipo que faltam achar / casas desconhecidas (o enunciado diz quantos existem: 8 poços, 4 inimigos, 4 teletransportes);
- **evidência:** para cada vizinha que sentiu o perigo sem explicação, `1/K`, onde K é o número de casas que podem ser a causa.

Com essa chance, a regra de risco é:

| Perigo possível | Aceita? | Motivo |
| --- | --- | --- |
| palmas (teletransporte) | sim | Não mata sozinho, só leva para outro lugar |
| passos (inimigo) | só com energia > 80 | Tira energia mas não pontos; com > 80 o agente aguenta o maior golpe e foge |
| brisa (poço) | só com chance < 20% | Morte na hora e -1000 |

A memória pode repetir uma percepção (`[brisa, brisa]` quando o vizinho tinha dois poços ao lado). O
`risco` passa a lista por `sort/2` para não contar duas vezes.

Entre as casas aceitáveis, escolhe a de menor risco = Σ peso × chance, com pesos poço 10, inimigo 3 e
teletransporte 1. Se nenhuma for aceitável, volta para [1,1] e sai com o ouro que tem.

Os dois parâmetros estão no topo da seção de risco de `agente.pl` (`aceita_risco/1` e `limite_chance/1`).
Também podem ser trocados pela linha de comando do `rodar_testes.py`.

## Experimentos

As tabelas de comparação abaixo registram os experimentos anteriores à correção do limite de energia
descrita no item 7 da evolução. Os resultados atuais da política escolhida estão na seção "Resultados"
e em `resultados/`.

### Qual risco aceitar

O mesmo agente rodou com políticas de risco diferentes, sempre com limite de chance de poço de 20%. Foram
10 partidas por mapa fornecido e os mesmos 200 mapas aleatórios:

| Riscos aceitos | Fácil | Médio | Difícil | Aleatórios: pontuação média | Ouro médio | Sobreviveu | 3 ouros |
| --- | --- | --- | --- | --- | --- | --- | --- |
| nenhum | 1234 | -25 | 1151 | 776,4 | 1,88 | 198/200 | 118/200 |
| teletransporte | 1234 | -25 | 1151 | 868,7 | 2,15 | 194/200 | 137/200 |
| **teletransporte + inimigo (escolhida)** | 1234 | **1130** | 1151 | **936,4** | 2,33 | 190/200 | 148/200 |
| tudo (até poço) | 1234 | 1130 | 1151 | 937,2 | 2,60 | 167/200 | 167/200 |

Quanto mais risco, mais ouro e menos sobrevivência. Ficamos com "teletransporte + inimigo":

- é a única que resolve o mapa médio sem arriscar poço às cegas;
- tem praticamente a mesma pontuação média de "tudo" (936 contra 937);
- sobrevive bem mais: 95% contra 83,5%, e o professor pediu para o agente não morrer.

Também testamos o limite de chance de poço:

| Limite | Mapa médio | Aleatórios: pontuação média | Sobreviveu |
| --- | --- | --- | --- |
| 0,05 | -23 (0 ouro) | 859,5 | 189/200 |
| 0,1 | 1130 | 924,9 | 190/200 |
| **0,2** | 1130 | **936,4** | 190/200 |
| 0,3 | 1130 | 933,8 | 189/200 |

Com 0,05 o mapa médio não é resolvido; de 0,1 a 0,3 a diferença é pequena. Ficamos com 0,2.

### Evolução do agente

1. **Só casas seguras (só a inferência do código-base).** Fácil: 3 ouros. Médio: 0 ouro (-23). Difícil: 1
   ouro (302). No médio, as casas em volta do início têm brisa, e o código-base não conclui nada.
2. **Percepção guardada e regra de diagnóstico em todas as casas visitadas.** O agente passou a confirmar
   poços longe da posição atual, como o poço em [1,3] do mapa médio. Difícil: 3 ouros (1151).
3. **Chance a priori na avaliação de risco.** Antes, uma brisa "já explicada" contava como chance 0. O
   agente morreu num mapa onde faltava 1 poço em 2 casas desconhecidas (chance real de 50%).
   Sobrevivência nos aleatórios: 188 → 192 (com teletransporte aceito).
4. **Aceitar risco de inimigo.** Médio: 3 ouros (1130).
5. **Fronteira alcançável, atravessar inimigo conhecido e teletransporte como saída de emergência.**
   Corrigiram casos em que o agente escolhia uma casa inalcançável e acabava num poço, ou ficava sem
   caminho para [1,1] porque havia um inimigo na porta.
6. **Revisão.** Três mudanças:
   - `observacao_certeza` saiu, porque podia criar poços falsos;
   - inimigo só é arriscado com energia > 80, e a fuga anda para frente quando dá;
   - percepções repetidas deixaram de contar duas vezes.

   Sobrevivência nos aleatórios: 187 → 190; média: 931,6 → 936,4.

7. **Limite de energia em toda aceitação de risco de inimigo.** A alternativa de chance baixa podia
   aceitar `passos` com energia ≤ 80 quando a primeira regra falhava. A condição de energia agora vem
   antes da escolha entre risco aceito e chance baixa. Quatro novos testes cobrem o limite de energia,
   a alternativa de chance baixa e a preservação do risco de teletransporte. Nas mesmas 290 partidas,
   os mapas fornecidos mantiveram os resultados; a média nos aleatórios passou de 936,4 para 937,0,
   com 190/200 sobreviventes e 148/200 saídas com três ouros.

## Mudanças no código-base do professor

| Arquivo | Mudança | Por quê |
| --- | --- | --- |
| `main.pl` | `executa_acao/1` aleatório foi removido; `main.pl` agora carrega `agente.pl` | Pedido do enunciado |
| `main.pl` | `percepcao/3` guardada em `atualiza_obs` | TELL da percepção (Aula 07.1), para a regra de diagnóstico |
| `main.pl` | `observacao_certeza` saiu de `atualiza_obs` | Podia marcar poço falso (ver "Inferência"); `deduz_perigos` faz a dedução certa |
| `main.pl` | `set_real` ao acordar de um teletransporte | Ao andar, o agente já percebia a casa onde entra; ao cair do teletransporte, não |
| `main.pl` | `ouros/1` e `sair/0` | Contar o ouro e a vitória por retorno a [1,1], que o enunciado diz não existir no código-base |
| `main.pl` | `carrega_mapa/1` | Trocar de mapa sem editar arquivo; também recarrega o ouro já pego |
| `gmap.py` | Mapa pela linha de comando, autoplay ligado, ouro, ações e status na tela, plano e casas OK desenhados | Requisitos de exibição e teste |
| `gmap.py` | A tecla M só redesenha | Antes ela chamava `verifica_player` de novo, e o inimigo dava dano extra |

O agente **nunca** consulta `tile/3` nem `adjacentes/1`. O mapa real só é lido pelo motor, para gerar
as percepções e aplicar dano. O `rodar_testes.py` também lê, só para anotar a causa da morte no relatório.


Teclas: **A** liga/desliga o automático; no manual, setas viram e andam e espaço pega. **M** mostra o
mapa real, só para conferência (o agente continua sem vê-lo).

Na tela:
- chão colorido = casa visitada;
- ícone colorido = perigo ou item com certeza;
- ícone cinza = possível;
- borda verde = casa OK ainda não visitada;
- ponto amarelo = caminho planejado.

Testes e registros:

```powershell
python rodar_testes.py                    # 30 partidas em cada mapa fornecido
python rodar_testes.py --aleatorios 200   # + 200 mapas sorteados com os mesmos elementos
python rodar_testes.py --risco palmas     # outra política de risco
python -m unittest discover tests         # testes do agente
```

Os resultados vão para `resultados/resumo.md` (tabela) e `resultados/partidas.csv` (uma linha por partida).

Para depurar no SWI-Prolog: `consult('main.pl').`, depois `executa_acao(X).`, a ação, `atualiza_obs, verifica_player.`
e `show_mem.`

## Arquivos

| Arquivo | O que faz |
| --- | --- |
| `agente.pl` | O agente: conhecimento, inferência, risco, busca em largura e `executa_acao/1` |
| `main.pl` | Motor do jogo do professor (ações, percepções, pontuação), com as mudanças da tabela acima |
| `mapa_facil.pl`, `mapa_medio.pl`, `mapa_dificil.pl` | Mapas fornecidos (`mapa.pl` = médio) |
| `gmap.py` | Interface Pygame do professor, com o HUD e o desenho do plano |
| `rodar_testes.py` | Partidas sem interface, mapas aleatórios e tabela de resultados |
| `tests/test_agente.py` | Testes do agente |
| `docs/roteiro_video.md` | Roteiro do vídeo e perguntas prováveis |

## Checklist do enunciado

| Requisito | Onde |
| --- | --- |
| Matriz 12 x 12, início em [1,1], 100 de energia | Mapas e `reset_game` (código-base) |
| Ações andar, virar à direita, virar à esquerda e pegar com os custos do enunciado | `main.pl` (código-base) |
| Sensores brisa, passos, palmas, brilho e reflexo | `atualiza_obs` e `set_real` (código-base) |
| Conhecimento, inferência e escolha da ação em Prolog | `agente.pl` |
| `executa_acao/1` aleatório trocado por regras, memória, risco e planejamento | `agente.pl`, seção "Escolha da acao" |
| Sem acesso ao mapa real | `agente.pl` (teste `test_agente_nao_consulta_o_mapa_real`) |
| Vitória ao voltar para [1,1] | `sair/0` em `main.pl` e regra 7 do agente |
| Pontuação durante e no fim da execução | `gmap.py` (HUD) |
| Carregar os mapas fornecidos sem editar a base | `carrega_mapa/1`, `python gmap.py <mapa>` |
| Testes com pontuação, ouro, sobrevivência e ações | `rodar_testes.py` e `resultados/` |
| Vídeo de até 10 min | Roteiro em `docs/roteiro_video.md` |

## Limitações

- **Chance aproximada:** a chance de perigo não é uma probabilidade exata, que exigiria enumerar todos os
  modelos possíveis (model checking da Aula 07.1). Ela erra principalmente quando duas observações se
  sobrepõem.
- **Teletransporte:** ao aceitar o risco de teletransporte, o agente pode acordar dentro de um poço
  (8 em 144 casas). Nenhuma regra evita isso.
- **Inimigo:** o motor tira energia a cada ação feita na casa do inimigo, inclusive ao virar. Se o agente
  arrisca uma casa que tem um D e a única saída segura está atrás, ele leva até três golpes e morre (um D
  mata em dois). Exigir chance de inimigo < 50% evita parte desses casos, mas, num teste com 200 mapas,
  baixou a média de 936 para 875 pontos (sobrevivência 190 → 194). Ficamos com a regra mais simples.
- **Quantidades:** a chance a priori usa as quantidades do enunciado (8 poços, 4 teletransportes). O
  `mapa_facil.pl` tem só 5 poços e 3 teletransportes, então ali a chance fica um pouco superestimada, o
  que deixa o agente mais cauteloso.
- **Giros:** a BFS minimiza passos, não giros. Ela só prefere seguir reto no desempate.
- **Mapa fixo:** num mapa fixo o agente é determinístico, então cada mapa fornecido dá sempre o mesmo
  resultado, a menos que entre num teletransporte ou leve dano. Por isso testamos também em mapas aleatórios.
- **Região sem saída:** depois de um teletransporte, o agente pode ficar numa região sem caminho seguro
  para [1,1]. Aí ele arrisca (regra 9).
