# Gramática Livre de Contexto

O código define uma classe chamada Gramática que serve para representar e transformar gramáticas livres de contexto a partir de uma entrada textual o construtor da classe inicializa conjuntos para armazenar variáveis e terminais define a variável inicial da gramática como None e cria um dicionário de regras usando defaultdict(list) .

```
class Gramática:
    def __init__(self, regras_str):
        self.variaveis = set()
        self.terminais = set()
        self.inicial = None
        self.regras = defaultdict(list)
        self._carregar_gramatica(regras_str)
```
O método remover_simbolos_inuteis elimina símbolos que não geram cadeias terminais ou que não são acessíveis a partir do símbolo inicial ele faz isso em duas etapas a primeira identifica símbolos que podem gerar cadeias terminais chamadas de geradores e a segunda percorre a gramática a partir do símbolo inicial para identificar os símbolos acessíveis após isso remove todas as produções e variáveis que não são geradoras nem acessíveis.

```
 def remover_simbolos_inuteis(self):
        geradores = set()
        alterado = True
        while alterado:
            alterado = False
            for A in self.regras:
                for prod in self.regras[A]:
                    if all(s in self.terminais or s in geradores for s in prod):
                        if A not in geradores:
                            geradores.add(A)
                            alterado = True

        self.regras = {A: [p for p in prods if all(s in self.terminais or s in geradores for s in p)]
                       for A, prods in self.regras.items() if A in geradores}
        self.variaveis = set(self.regras.keys())

        acessiveis = set()
        fila = [self.inicial]

        while fila:
            atual = fila.pop()
            if atual not in acessiveis:
                acessiveis.add(atual)
                for prod in self.regras.get(atual, []):
                    for simbolo in prod:
                        if simbolo in self.variaveis and simbolo not in acessiveis:
                            fila.append(simbolo)

        self.regras = {A: [p for p in prods] for A, prods in self.regras.items() if A in acessiveis}
        self.variaveis = set(self.regras.keys())
```

O método remover_producoes_vazias encontra todas as variáveis anuláveis ou seja aquelas que podem gerar `ε` diretamente ou indiretamente e para cada produção que contenha essas variáveis gera todas as combinações possíveis omitindo essas variáveis o objetivo é eliminar produções com `ε` sem alterar a linguagem da gramática se a variável inicial for anulável uma nova produção com `ε` é adicionada explicitamente

```
    def remover_producoes_vazias(self):
        anulaveis = set()
        alterado = True
        while alterado:
            alterado = False
            for A, prods in self.regras.items():
                for prod in prods:
                    if all(s in anulaveis or s == 'ε' for s in prod):
                        if A not in anulaveis:
                            anulaveis.add(A)
                            alterado = True

        novas_regras = defaultdict(list)

        for A, prods in self.regras.items():
            for prod in prods:
                if prod == ['ε']:
                    continue
                subconjuntos = gerar_substituicoes_vazias(prod, anulaveis)
                for p in subconjuntos:
                    if not p:
                        novas_regras[A].append(['ε'])
                    elif p not in novas_regras[A]:
                        novas_regras[A].append(p)

        if self.inicial in anulaveis:
            novas_regras[self.inicial].append(['ε'])

        self.regras = novas_regras
```

Ao final do código a função gerar_substituicoes_vazias auxilia a remoção de produções vazias gerando todas as combinações possíveis de uma produção omitindo símbolos anuláveis ela retorna todas essas variações para que possam ser usadas no método `remover_producoes_vazias`

```
def gerar_substituicoes_vazias(prod, anulaveis):
    indices = [i for i, s in enumerate(prod) if s in anulaveis]
    subconjuntos = chain.from_iterable(combinations(indices, r) for r in range(1, len(indices)+1))
    resultados = []
    for subconjunto in subconjuntos:
        nova = [s for i, s in enumerate(prod) if i not in subconjunto]
        if nova not in resultados:
            resultados.append(nova)
    resultados.append(prod)
    return resultados
```

O método substituir_unitarias remove produções do tipo `A -> B` onde `B` é outra variável o algoritmo percorre as produções unitárias para encontrar o fechamento transitivo de cada variável ou seja todas as variáveis que podem ser alcançadas por meio de produções unitárias e então copia para `A` todas as produções não unitárias dessas variáveis

```
    def substituir_unitarias(self):
        novas_regras = defaultdict(list)

        for A in self.regras:
            fecho = set()
            fila = [A]

            while fila:
                atual = fila.pop()
                for prod in self.regras.get(atual, []):
                    if len(prod) == 1 and prod[0] in self.variaveis:
                        B = prod[0]
                        if B not in fecho:
                            fecho.add(B)
                            fila.append(B)

            for B in fecho | {A}:
                for prod in self.regras.get(B, []):
                    if not (len(prod) == 1 and prod[0] in self.variaveis):
                        if prod not in novas_regras[A]:
                            novas_regras[A].append(prod)

        self.regras = novas_regras
```

O método para_fnc transforma a gramática na forma normal de Chomsky que exige que todas as produções sejam da forma `A -> BC` ou `A -> a` inicialmente ele substitui os terminais por variáveis se estiverem acompanhados de outros símbolos usando variáveis auxiliares como `X1`, `X2` etc depois divide qualquer produção com mais de dois símbolos em múltiplas produções binárias criando novas variáveis intermediárias para manter a forma exigida

```
def para_fnc(self):
        novas_regras = defaultdict(list)
        novos_simbolos = {}
        contador = 1

        def nova_variavel():
            nonlocal contador
            while True:
                var = f"X{contador}"
                contador += 1
                if var not in self.variaveis and var not in self.terminais:
                    return var

        for A in self.regras:
            for prod in self.regras[A]:
                if len(prod) == 1 and prod[0] in self.terminais:
                    novas_regras[A].append(prod)
                else:
                    nova_prod = []
                    for s in prod:
                        if s in self.terminais:
                            if s not in novos_simbolos:
                                X = nova_variavel()
                                novos_simbolos[s] = X
                            nova_prod.append(novos_simbolos[s])
                        else:
                            nova_prod.append(s)
                    novas_regras[A].append(nova_prod)

        for terminal, var in novos_simbolos.items():
            novas_regras[var].append([terminal])
            self.variaveis.add(var)

        final_regras = defaultdict(list)
        for A in novas_regras:
            for prod in novas_regras[A]:
                while len(prod) > 2:
                    X = nova_variavel()
                    self.variaveis.add(X)
                    final_regras[X].append(prod[1:])
                    prod = [prod[0], X]
                final_regras[A].append(prod)

        self.regras = final_regras
```


O método para_fng transforma a gramática na forma normal de Greibach onde cada produção começa com um terminal seguido de zero ou mais variáveis ele percorre as variáveis em ordem alfabética e sempre que encontra uma produção iniciada por uma variável anterior reescreve essa produção usando as produções da variável anterior se encontrar recursão à esquerda direta aplica o procedimento clássico de eliminação criando uma nova variável e reescrevendo as produções de forma a remover a recursividade

```
def para_fng(self):
      variaveis_ordenadas = sorted(list(self.variaveis))
      novas_regras = dict(self.regras)

      for i, Ai in enumerate(variaveis_ordenadas):
          novas_producoes = []
          for prod in novas_regras.get(Ai, []):
              if prod and prod[0] in self.variaveis:
                  Aj = prod[0]
                  j = variaveis_ordenadas.index(Aj)
                  if j < i:
                      for gamma in novas_regras.get(Aj, []):
                          novas_producoes.append(gamma + prod[1:])
                  else:
                      novas_producoes.append(prod)
              else:
                  novas_producoes.append(prod)
          novas_regras[Ai] = novas_producoes

          diretas = []
          indiretas = []

          for prod in novas_regras[Ai]:
              if prod and prod[0] == Ai:
                  diretas.append(prod[1:])
              else:
                  indiretas.append(prod)

          if diretas:
              Ai_linha = Ai + "'"
              self.variaveis.add(Ai_linha)
              novas_regras[Ai] = []
              for beta in indiretas:
                  novas_regras[Ai].append(beta + [Ai_linha])
              novas_regras[Ai_linha] = []
              for alpha in diretas:
                  novas_regras[Ai_linha].append(alpha + [Ai_linha])
              novas_regras[Ai_linha].append(['ε'])

      final_regras = defaultdict(list)
      for A in novas_regras:
          for prod in novas_regras[A]:
              if prod and prod[0] in self.variaveis:
                  for alt in novas_regras.get(prod[0], []):
                      final_regras[A].append(alt + prod[1:])
              else:
                  final_regras[A].append(prod)

      self.regras = final_regras
```

O método fatorar_a_esquerda é a melhoria que reorganiza as produções para resolver ambiguidade quando há várias produções com prefixos comuns por exemplo se `A -> abc | abd` ele cria um novo símbolo para representar o sufixo e reescreve como `A -> abF1`, `F1 -> c | d` isso evita conflitos no momento da derivação e torna a gramática adequada para parsers do tipo preditivo

```
    def fatorar_a_esquerda(self):
        novas_regras = defaultdict(list)
        contador = 1

        def nova_variavel():
            nonlocal contador
            while True:
                var = f"F{contador}"
                contador += 1
                if var not in self.variaveis:
                    return var

        for A, producoes in self.regras.items():
            prefixos = defaultdict(list)
            for prod in producoes:
                if prod:
                    prefixos[prod[0]].append(prod)

            if all(len(v) == 1 for v in prefixos.values()):
                novas_regras[A].extend(producoes)
            else:
                for simbolo, grupo in prefixos.items():
                    if len(grupo) == 1:
                        novas_regras[A].append(grupo[0])
                    else:
                        novo_nome = nova_variavel()
                        self.variaveis.add(novo_nome)
                        novas_regras[A].append([simbolo, novo_nome])
                        for prod in grupo:
                            sufixo = prod[1:] if len(prod) > 1 else ['ε']
                            novas_regras[novo_nome].append(sufixo)

        self.regras = novas_regras
```

O método remover_recursao_esquerda elimina recursão à esquerda direta ao identificar produções como `A -> Aα | β` onde `β` não começa com `A` nesse caso ele cria uma nova variável como `R1` para separar as partes recursivas e não recursivas a produção é reescrita como `A -> βR1` e `R1 -> αR1 | ε` garantindo que as derivações comecem por cadeias não recursivas

```
    def remover_recursao_esquerda(self):
        novas_regras = defaultdict(list)
        contador = 1

        def nova_variavel():
            nonlocal contador
            while True:
                var = f"R{contador}"
                contador += 1
                if var not in self.variaveis:
                    return var

        for A, prods in self.regras.items():
            diretas = []
            indiretas = []

            for prod in prods:
                if prod and prod[0] == A:
                    diretas.append(prod[1:])
                else:
                    indiretas.append(prod)

            if diretas:
                novo = nova_variavel()
                self.variaveis.add(novo)
                for beta in indiretas:
                    novas_regras[A].append(beta + [novo])
                for alpha in diretas:
                    novas_regras[novo].append(alpha + [novo])
                novas_regras[novo].append(['ε'])
            else:
                novas_regras[A].extend(prods)

        self.regras = novas_regras
```

No final o código define uma gramática de entrada com as regras `S -> aAa | bB` e `A -> a | aA` cria um objeto da classe `Gramática` exibe a gramática original e em seguida aplica em ordem os métodos de remoção de símbolos inúteis remoção de produções vazias substituição de produções unitárias transformação para FNC transformação para FNG fatoração à esquerda e remoção de recursão à esquerda exibindo o resultado após cada etapa de transformação tudo isso permite acompanhar a evolução da gramática desde sua forma original até uma versão simplificada e estruturada para análises ou implementações de análise sintática em compiladores

```
entrada = """
S -> aAa | bB
A -> a | aA
"""

g = Gramática(entrada)
g.exibir("Gramática Original")

g.remover_simbolos_inuteis()
g.exibir("Após remoção de símbolos inúteis")

g.remover_producoes_vazias()
g.exibir("Após remoção de produções vazias")

g.substituir_unitarias()
g.exibir("Após substituição de produções unitárias")

g.para_fnc()
g.exibir("Após conversão para FNC")

g.para_fng()
g.exibir("Após conversão para FNG")

g.fatorar_a_esquerda()
g.exibir("Após fatoração à esquerda")

g.remover_recursao_esquerda()
g.exibir("Após remoção de recursão à esquerda")
```
