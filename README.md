# Rumo: o caminho para a sua casa e pela sua cidade

Assistente de terminal em Python feito para a Global Solution 2026.2 (CitizenTech, case Citizen Assist AI), na matéria Pensamento Computacional e Automação com Python, FIAP, turma 1CCPY (1º ano de Ciência da Computação).

## Integrantes

| Nome | RM |
|---|---|
| Jair Ferreira dos Santos Neto | 569682 |
| Arthur dos Santos Bezerra | 569721 |
| Carlos Henrique Fratezi | 571792 |

## Público e problema

O Rumo atende **moradores da cidade de São Paulo** com dúvidas sobre:

- **Moradia:** Minha Casa Minha Vida, cadastro da COHAB-SP, auxílio aluguel, Locação Social e regularização do imóvel.
- **Transporte:** Bilhete Único (como tirar, recarga, bloqueio), gratuidades para pessoa idosa, pessoa com deficiência e estudante, integração e linhas de ônibus.
- **Energia em casa:** Tarifa Social (Luz do Povo, com o caminho do CadÚnico), Desconto Social e energia solar (telhado próprio ou assinatura).

O problema: as regras desses serviços ficam espalhadas em vários sites oficiais, com linguagem técnica. Muita gente não sabe se tem direito, quais documentos levar nem onde ir. O Rumo responde em linguagem simples e sempre mostra de onde a informação veio, quando foi verificada e um aviso de que **não é canal oficial**.

## ODS relacionados

- **ODS 11 (Cidades e comunidades sustentáveis):** acesso à moradia e ao transporte público.
- **ODS 10 (Redução das desigualdades):** orientação sobre direitos de pessoas idosas, pessoas com deficiência e famílias de baixa renda.
- **ODS 7 (Energia limpa e acessível):** Tarifa Social de energia e simulação de economia na conta de luz.

## Como executar

É preciso ter o Python 3.10 ou mais novo instalado. O Rumo não usa nenhuma biblioteca externa.

```bash
git clone https://github.com/<usuario>/citizen-assist-ai.git
cd citizen-assist-ai
python main.py
```

Menu principal:

```
1 - Fazer uma pergunta
2 - Ver assuntos por categoria
3 - Ver indicadores dos atendimentos
4 - Exportar atendimentos para CSV
5 - Simular economia na conta de luz
0 - Sair
```

## Estrutura

```
citizen-assist-ai/
├── main.py               # ponto de entrada: cria a base e o assistente
├── orientacao.py         # classe Orientacao (uma orientação + exibir())
├── base_conhecimento.py  # classe BaseConhecimento + normalizar() + pontuação
├── assistente.py         # classe Assistente (menus, histórico, indicadores, CSV)
├── energia/              # simulador de economia na conta de luz (Energias Renováveis)
└── data/
    ├── base_conhecimento.json   # 17 orientações em 3 categorias
    ├── historico.json           # atendimentos registrados
    └── atendimentos.csv         # exportação do histórico
```

### Classes e composição

- `Orientacao`: guarda os campos de uma orientação (`id, categoria, titulo, palavras_chave, resposta, documentos, onde, fonte, verificado_em`). O método `exibir()` mostra a orientação com a fonte, a data e o aviso.
- `BaseConhecimento`: guarda `caminho` e `orientacoes` (uma lista de objetos `Orientacao`). Tem os métodos `carregar()`, `categorias()`, `por_categoria()`, `pontuar()` e `buscar()`.
- `Assistente`: guarda `nome`, `base` (um objeto `BaseConhecimento`) e `historico`. Tem os métodos `perguntar()`, `navegar()`, `registrar()`, `indicadores()` e `exportar_csv()`.

**Composição:** o `Assistente` tem uma `BaseConhecimento`, e a `BaseConhecimento` tem vários objetos `Orientacao`. No `main.py`:

```python
base = BaseConhecimento("data/base_conhecimento.json")
base.carregar()
assistente = Assistente("Rumo", base)
assistente.iniciar()   # abre o menu, que chama perguntar(), navegar() etc.
```

## Regra de pontuação

1. **Normalização:** a função `normalizar()` deixa a pergunta e as palavras-chave em minúsculas, sem acentos (usando o dicionário `ACENTOS`) e sem pontuação.
2. **+3 pontos** para cada palavra-chave da orientação que aparece na pergunta. A comparação é feita por palavra inteira, então palavras-chave com mais de uma palavra, como "bilhete unico", também funcionam.
3. **+1 ponto** para cada palavra da pergunta com 4 letras ou mais que aparece no título da orientação. Cada palavra conta só uma vez. Palavras comuns que não indicam assunto (`para, pelo, pela, mais, esta, isso, sobre`) são ignoradas, porque somavam pontos para títulos que não tinham nada a ver com a pergunta.
4. **Confiança:** 6 pontos ou mais = **ALTA**; de 3 a 5 = **MÉDIA**; menos de 3 = **não encontrei**. Nesse caso, o Rumo sugere outras palavras e as categorias.
5. `buscar()` devolve uma lista de **tuplas `(pontos, orientacao)`** em ordem decrescente. O Rumo mostra a melhor e até 2 alternativas, que só aparecem se também tiverem confiança MÉDIA ou ALTA.

**Exemplo real:** "Tenho 62 anos, pago ônibus?"

- normalizada: `tenho 62 anos pago onibus`
- orientação TRA01 (Ônibus grátis para pessoa idosa): palavras-chave `anos` (+3) e `pago onibus` (+3); palavras no título: `anos` (+1) e `onibus` (+1). Total: **8 pontos, ALTA**.
- alternativa TRA05 (Pessoa com Deficiência): palavra-chave `pago onibus` (+3). Total: **3 pontos, MÉDIA**.

Os pesos ficam em constantes no começo de `base_conhecimento.py` (`PESO_PALAVRA_CHAVE`, `PESO_TITULO`, `LIMITE_ALTA`, `LIMITE_MEDIA`) e podem ser ajustados.

## Regras de segurança e entradas inválidas

- O Rumo **nunca pede** nome, CPF, telefone ou e-mail.
- Uma pergunta com **8 dígitos ou mais** ou com **@** é recusada e **não é registrada**.
- Opções inválidas, respostas vazias, Ctrl+C e Ctrl+D não encerram o programa.
- Todas as telas de navegação têm a opção "0 - Voltar".
- Se `base_conhecimento.json` estiver ausente ou corrompido, o programa avisa e continua aberto. Orientações sem algum campo obrigatório são ignoradas com aviso.
- Se `historico.json` estiver corrompido, um novo histórico é iniciado.

## Histórico, indicadores e CSV

Cada atendimento é salvo em `data/historico.json` com `data, origem (pergunta ou categoria), pergunta, id_orientacao, encontrou, ajudou, tentativas, tempo` (segundos, medidos com `time.time()` e arredondados com `round(..., 1)`) e `facilidade`.

- **Tentativas:** no modo pergunta, um atendimento pode ter várias tentativas até a pessoa dizer que a dúvida foi resolvida, ou até ela desistir digitando 0. Nesse caso, o atendimento é registrado com `ajudou = False`.
- **Facilidade:** ao sair, o Rumo pede uma nota de 1 a 5. A nota é gravada nos atendimentos daquela execução.
- **Indicadores (opção 3):** % de respostas encontradas, % que ajudaram, tempo médio, tentativas por atendimento, facilidade média e lista de dúvidas sem solução.
- **CSV (opção 4):** gera `data/atendimentos.csv` com `csv.DictWriter`. Usamos `;` como separador e codificação `utf-8-sig` para o arquivo abrir certo no Excel em português.

## Integrações

- **Energias Renováveis:** a categoria "Energia em casa" está na base de conhecimento, e a opção 5 do menu chama `energia.calculadora_energia.menu_energia()`, o simulador de economia na conta de luz da matéria de Energias Renováveis. A chamada fica dentro de `try/except`: se o módulo faltar ou der erro, o Rumo avisa e volta ao menu.
### SERS: Energia em casa (responsável: Jair, RM 569682)

Toda a contribuição de Soluções em Energias Renováveis está em `energia/` (ver `energia/README.md`). É uma calculadora em Python que compara eficiência + Tarifa Social com energia solar (telhado e assinatura) para famílias de baixa renda em SP. Ela usa a tarifa da Enel SP (ANEEL REH 3.596/2026), o fator de CO2 do MCTI e premissas com fonte em `energia/dados/premissas_energia.json`, e tem testes. No assistente: opção 5 do menu e orientações ENE01 a ENE03.

```bash
python -m unittest discover -s energia/testes -t .   # 15 testes
python -m energia.calculadora_energia --relatorio    # gera energia/analise/resultados.md
```

- **Vídeo pitch:** a demonstração "tenho 62 anos, pago ônibus?" do pitch usa esta mesma base e esta mesma regra de pontuação.

## Fontes oficiais (verificadas em 16/11/2026; TRA07 e TRA08 em 18/11/2026)

| Orientação | Fonte |
|---|---|
| MOR01 Minha Casa Minha Vida | Ministério das Cidades, gov.br/cidades; Portaria MCID nº 333/2026 |
| MOR02 Cadastro COHAB-SP | cadastro.cohab.sp.gov.br |
| MOR03 Auxílio aluguel (SEHAB) | prefeitura.sp.gov.br/web/habitacao |
| MOR04 Auxílio aluguel mulher | Prefeitura de SP, Secretaria Municipal de Direitos Humanos e Cidadania |
| MOR05 Escritura na Mão | Prefeitura de SP, SEHAB; Lei Municipal 17.859/2022 |
| MOR06 Locação Social / Pode Entrar | Prefeitura de SP / COHAB-SP |
| TRA01 Pessoa idosa | SPTrans, Perguntas e respostas; Decreto Municipal 62.057/2022 |
| TRA02 Bilhete Único comum | bilheteunico.sptrans.com.br/comum |
| TRA03 Integração e tarifas | sptrans.com.br/tarifas |
| TRA04 Estudante | estudante.sptrans.com.br |
| TRA05 Pessoa com deficiência | SPTrans, Perguntas e respostas (Bilhete PcD) |
| TRA06 Linhas e Olho Vivo | olhovivo.sptrans.com.br |
| TRA07 Recarga sem sair de casa | sptrans.com.br/app (aplicativos credenciados); bilheteunico.sptrans.com.br (loja virtual) |
| TRA08 Bilhete bloqueado ou perdido | SPTrans, Perguntas e respostas e nota sobre créditos de cartões bloqueados; Portal SP156 |
| ENE01 Tarifa Social / Luz do Povo e CadÚnico | gov.br/mme/pt-br/luzdopovo; Lei 15.235/2025; Prefeitura de SP (SMADS), CadÚnico |
| ENE02 Desconto Social | gov.br/mme (Luz do Povo) |
| ENE03 Energia solar | Lei 14.300/2022; ANEEL REN 1.000/2021; ANEEL REH 3.596/2026; cálculo da equipe em `energia/` (estimativa) |

Quando não encontramos um valor confirmado em fonte oficial, como o preço atual da tarifa ou datas de inscrição da Locação Social, a orientação **não traz o número** e indica onde consultar.

## Resultado do teste com usuários

Teste feito em 18/11/2026 com **4 pessoas de fora da equipe** (familiares), sem registrar nome nem dado delas. Cada pessoa recebeu situações do dia a dia (por exemplo, "você acabou de completar 60 anos: veja se ainda precisa pagar ônibus") e usou o Rumo sozinha, por pergunta livre e pela navegação por categorias. Os registros estão em `data/historico.json` e `data/atendimentos.csv`.

| Indicador | Resultado |
|---|---|
| Pessoas que testaram | 4 |
| Atendimentos registrados | 12 (8 por pergunta, 4 por categoria) |
| Respostas encontradas | 100% |
| Ajudou | 100% |
| Tempo médio | 23,9 s |
| Tentativas por atendimento | 1,4 |
| Facilidade média (1 a 5) | 3,5 (notas 5, 4, 3 e 2) |

**Dúvidas sem solução:** nenhuma ficou marcada como "não ajudou". Mas, olhando o histórico, três atendimentos foram resolvidos só em parte:

- "Você quer colocar crédito no Bilhete Único sem sair de casa": **5 tentativas e 108,7 s**. A base não tinha orientação sobre recarga, e a pessoa acabou aceitando a TRA02 (como tirar o Bilhete Único).
- "Seu Bilhete Único foi bloqueado": caiu na TRA02, que não explica bloqueio.
- "Sua conta de luz está atrasada e você tem medo de que cortem a energia": caiu na ENE01 (Tarifa Social), que ajuda a pagar menos, mas não explica as regras de corte.

A pessoa que deu nota 2 foi justamente a que fez as duas perguntas sobre o Bilhete Único.

**Melhoria feita a partir do teste:** criamos duas orientações novas na categoria Transporte, com fonte oficial da SPTrans:

- **TRA07, "Recarga do Bilhete Único sem sair de casa"**, com aplicativos credenciados e loja virtual.
- **TRA08, "Bilhete Único bloqueado, perdido ou roubado"**, com os motivos de bloqueio, a justificativa pelo atendimento digital e o aviso de que os créditos não se perdem.

Também trocamos a palavra-chave "sair de casa" da MOR04 por "preciso sair de casa", porque ela fazia a orientação sobre violência doméstica aparecer como alternativa na pergunta de recarga. Resultado, com as mesmas frases do teste:

| Pergunta do teste | Antes | Depois |
|---|---|---|
| "...colocar crédito no Bilhete Único sem sair de casa..." | TRA02, 5 tentativas | **TRA07, 17 pontos, ALTA** |
| "Seu Bilhete Único foi bloqueado..." | TRA02 | **TRA08, 9 pontos, ALTA** |

A dúvida sobre corte de energia por atraso ficou registrada como limitação.

## Limitações

- A busca compara palavras exatas, depois de normalizadas. Sinônimos que não estão nas palavras-chave e erros de digitação não são reconhecidos: em "pago onibos" a palavra-chave "pago onibus" não soma pontos.
- Palavras curtas e muito gerais (como "anos") ajudam a achar perguntas reais, mas podem somar pontos em perguntas de outro assunto.
- A base tem 17 orientações e só vale para a cidade de São Paulo. Ainda não cobre, por exemplo, as regras de corte de energia por conta atrasada (dúvida que apareceu no teste). As regras mudam, então cada orientação traz a data de verificação.
- O Rumo não é canal oficial, não consulta dados pessoais nem faz pedidos em nome do cidadão.

## Vídeo técnico

Link (YouTube, não listado): https://youtu.be/FkdzqQNu1mY
