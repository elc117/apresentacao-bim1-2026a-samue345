## DOCUMENTAÇÃO

O porgrama basicamente implementa um labirinto em primeira pessoa. O programa começa em um estado inicial que contém o mapa, a posição do jogador, a direção do jogador e asm teclas presionadas. O jogador pode andar pelo labirinto usando as teclas WASD.

```main :: IO ()
main =
  activityOf
    (State
      { worldMap = parseMap testMap
      , playerPos = (1.5,1.5)
      , playerDir = (1,1)
      , keysPressed = S.empty
      })
    handle
    render
```
O programa inicia na função main que chama apenas a função activityOf da biblioteca CodeWorld. A função activityOf é responsável por criar um programa interativo que começa com um estado inicial, atualiza esse estado em resposta a eventos e o desenha como uma imagem.

a função tem três parâmetros:

world. estado inicial, no nosso caso seria o state.

Event -> world -> world, Função de atualização

world -> Picture. Uma função que converte um World em um Picture, usado para exibir o programa na tela.

Para mais informações sobre como a função funciona está aqui o link da DOC: https://cw21.brbytes.org/cw21/doc/Standard.html

## Map

```
testMap ::[String]
testMap =
  [ "111111111111111111111111"
  , "100000100000000010000001"
  , "100000100022000010000001"
  , "100300100022000010000001"
  , "100000100000000010000001"
  , "100000100000000010000001"
  , "100111111000011110000001"
  , "100000000000000000000001"
  , "100000000000000000000001"
  , "100030000003000000030001"
  , "100000000000000000000001"
  , "100000000000000000000001"
  , "111111111111111111111111"
  ]
```
Representa o mapa do jogo em formato de texto. TestMap é umn array de strings em que cada string é uma linha do mapa, e cada caractere é uma célula: '0' → espaço vazio '1' → parede '2', '3' → outros tipos de parede
 
 ```
    type WallType = Int
    type Map = A.Array (Int, Int) WallType
```

 WallType: Define o tipo de uma célula do mapa. É um número inteiro que representa o tipo de parede ou espaço.
 Map: Define o mapa como um array indexado por coordenadas (x, y). Cada posição retorna um WallType.

 ```
parseMap :: [String] -> Map
parseMap rows =
  A.array ((0,0), (w-1,h-1)) (concat (zipWith parseRow [0..] rows))
 where
  w = length (head rows)
  h = length rows
  parseRow j row = zipWith (parseCell j) [0..] row
  parseCell j i cell = ((i,j), read [cell])
```
Essa função converte o mapa que está como uma lista de strings ([String]) para um array indexado por coordenadas (x, y).

Cada caractere das strings vira um número (WallType) associado a uma posição no mapa.

O parâmetro rows é o testMap, onde cada string representa uma linha e cada caractere representa uma célula.

Primeiro, são calculadas as dimensões do mapa, w é a largura e h é a altura.

Depois, A.array ((0,0), (w-1,h-1)) define o tamanho do array que será criado.

Em seguida, o código percorre o mapa.
O zipWith parseRow [0..] rows associa cada linha a um índice j (posição vertical). O zipWith é uma função que percorre duas listas ao mesmo tempo e aplica uma função aos elementos correspondentes. Aqui, ele pega os índices [0..] e as linhas do mapa, chamando parseRow j row para cada par.

Para cada linha, parseRow j row percorre os caracteres usando zipWith (parseCell j) [0..] row, associando cada caractere a um índice i (posição horizontal). Nesse caso, o zipWith faz a mesma coisa: combina os índices com os caracteres da linha.

A função parseCell j i cell transforma cada caractere em um par ((i,j), valor), onde o valor é o número correspondente ao caractere.

Como cada linha gera uma lista, concat é usado para juntar tudo em uma única lista de células.

Por fim, essa lista é passada para A.array, que cria o mapa final indexado por (x, y)

## State
```
data State = State 
  { worldMap :: !Map 
  , playerPos :: !Vector
  , playerDir :: !Vector
  , keysPressed :: !(S.Set T.Text)
  }
  deriving (Show)
```
Define o estado completo do jogo. O state é uma estrutura de dados chamada Algebraic Data Type, mais especificamente é um tipo com um único construtor (State) e campos nomeados. Nele contém:

WorldMap: Mapa do jogo convertido para um array indexado por coordenadas (x,y).
playerPos: Posição do jogador
playerDir: Direção do jogador
keysPressed: Teclas atualmente precionadas. De início ela recebe "empty", ou seja, nenhuma tecla é pressionada no início do programa.

## Vetores
```
i2d :: Int -> Double
i2d = fromIntegral

```
Converte o i2d de inteiro para Double

```
normalized :: Vector -> Vector
normalized v = if len < 1e-6 then (0,0) else scaledVector (1 / len) v
 where
  len = vectorLength v
```
função responsável por normalizar um vetor.
A função recebe  um vetor e retorna outro vetor com a mesma direção, mas com tamanho igual a 1. o vetor for muito pequeno (próximo de zero), retorna (0,0).

vectorLength v calcula o tamanho do vetor

scaledVector (1 / len) v ajusta o tamanho para 1

```
angleBetween :: Vector -> Vector -> Double
angleBetween u@(ux, uy) v@(vx, vy) = atan2 det dot
 where
  det = ux * vy - vx * uy
  dot = dotProduct u v
```
A função calcula o ângulo entre dois vetores. Ela Recebe dois vetores e retorna o ângulo entre eles em radianos.

dotProduct u v mede o alinhamento entre os vetores

det = ux * vy - vx * uy mede a rotação entre eles

atan2 det dot combina os dois para obter o ângulo correto


## Event Handling
```
handle :: Event -> State -> State
handle e w@(State {..}) = handle' e
 where
  handle' (TimePassing dt) =
    w { playerPos = playerPos `vectorSum` scaledVector (2*dt) speed }
   where
    speed = normalized $
      (keyToDir "W" playerDir)
      `vectorSum` (keyToDir "S" (scaledVector (-1) playerDir))
      `vectorSum` (keyToDir "A" (rotatedVector (pi/2) playerDir))
      `vectorSum` (keyToDir "D" (rotatedVector (-pi/2) playerDir))
    keyToDir k dir =
      if S.member k keysPressed then dir else (0,0)
  handle' (PointerMovement (x, _)) =
      w { playerDir = rotatedVector (-x * pi / 10) (0, 1) }
  handle' (KeyPress k) = w { keysPressed = S.insert k keysPressed }
  handle' (KeyRelease k) = w { keysPressed = S.delete k keysPressed }
  handle' _ = w
```
Essa função é responsável por lidar com os eventos do jogo.

Recebe um evento (teclado, mouse ou tempo) e o estado atual, e retorna um novo estado atualizado.

Basicamente, ela controla o movimento e a direção do jogador.

Quando o tempo passa (TimePassing dt), move o jogador de acordo com as teclas pressionadas (W, A, S, D).

Usa normalized para manter a velocidade constante, mesmo na diagonal.

Quando o mouse se move (PointerMovement), atualiza a direção do jogador.

Quando uma tecla é pressionada (KeyPress), adiciona ao conjunto de teclas ativas.

Quando uma tecla é solta (KeyRelease), remove do conjunto.

Outros eventos não alteram o estado.

## Fontes

SeroKell -  https://serokell.io/blog/algebraic-data-types-in-haskell
GitHub da disciplina: https://liascript.github.io/course/?https://raw.githubusercontent.com/AndreaInfUFSM/elc117-2026a/main/classes/03/README.md#24
documentação do codeWorld - https://cw21.brbytes.org/doc/

## Video da execução do programa
<img src="./video-pratica.gif" width="600">
