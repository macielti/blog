Title: Transformação/Coerção de valores com base em Schemas - Clojure
Date: 2025-11-13
Tags: Clojure, Schema, Coercion, Data Validation

Compartilhando um pouco do que sei sobre como se dá o processo de coerção de uma estrutura de dados usando um schema
como base.

### 1. Conceitos Fundamentais

### 1.1 Schemas

É uma notação utilizada pra descrever o formato de uma estrutura de dados. Quando trabalhamos com o Clojure, é muito
difundido no mercado a biblioteca [Plumatic/Prismatic Schema](https://github.com/plumatic/schema), ela fornece uma série
de helpers para facilitar a composição dos formatos de dados, o que chamamos de schemas, e também provê meios para que
possamos aplicar validações dos valores contra os formatos esperados, seja na validação da estrutura de dados de entrada
e saída de uma função, até mesmo o body de requisição ou de resposta de um determinado endpoint HTTP.

Exemplo do schema de um mapa (estrutura de dados) que representa uma pessoa, no exemplo definimos quais as chaves e os
tipos esperados para os valores:

```Clojure
(require '[schema.core])

(def ColorEnum (schema.core/enum :red :green :blue))

(schema.core/defschema Person
                       {:name           schema.core/String
                        :favorite-color ColorEnum})
```

No schema acima definimos que esperamos uma chave `:name` onde o valor é do tipo String, e uma chave `:favorite-color`
onde o valor esperado é um enum de keywords com três opções de possíveis valores.

Você pode recorrer a [documentação da biblioteca](https://github.com/plumatic/schema?tab=readme-ov-file#meet-schema)
para encontrar exemplos de como compor schemas e de como validar os mesmos com a descrição dos cenários de sucesso e
falhas na validação.

### 2. Transformação/Coerção de valores com base em um determinado schema

A documentação da [Plumatic/Prismatic Schema](https://github.com/plumatic/schema) que é a lib que utilizamos como base
para o mecanismos de validação e transformação comumente usada em aplicações desenvolvidas em Clojure, descreve o
processo de Coercion como sendo o mesmo que uma validação de um schema, mas adicionando um passo anterior a validação, a
transformação aplicada no mapa de entrada usando um schema base para a transformação, após a coerção dos valores é
aplicada a validação do dado retornado pela transformação com base no schema indicado.

1. Transformação do dado de entrada usando como fôrma um determinado schema.
2. Validação da saída da transformação com base no schema especificado.

Vamos ver na prática, como podemos fazer a transformação de um dado com base em um schema.
Para o nosso exemplo ilustrativo vamos criar um `FOO BAR coercer`:

1. Esperamos de entrada um mapa que contenha uma chave qualquer que aponte para um valor "FOO". As chaves e valores
   devem ser do tipo String.
2. Na etapa de transformação esperamos que as chaves do tipo String sejam convertidas para Keyword. E para os valores do
   mapa esperamos que todas as Strings "FOO" sejam substituídas por "FOO BAR".
3. No schema de saída esperamos um mapa com uma única Keyword como chave e que ela possua como valor a String  "FOO
   BAR".

Começamos com a definição dos exemplos que vamos usar de input para o processo de transformação e validação (coercion):

```Clojure
(def valid-foor-bar-json-input {"test" "FOO"})
(def invalid-foor-bar-json-input {"test-i" "Blue Pen"})
```

Configuração do mecanismo de coerção/transformação: Os Matchers para o processo de transformação/coerção, são nada mais
nada menos que mapas onde indicamos como cada tipo de dado deve ser adaptado a partir de um tipo X para um tipo Y.

```Clojure
(require '[schema.core])
(require '[schema.coerce])

(def FooBar (schema.core/eq "FOO BAR"))

(def foo-bar-coercions
  {FooBar              (fn [input] (if (= input "FOO") "FOO BAR" input))
   schema.core/Keyword (fn [input] (if (string? input) (keyword input) input))})

(defn foo-bar-matcher
      [schema]
      (foo-bar-coercions schema))

(schema.core/defschema FooBarResult
                       {schema.core/Keyword FooBar})

(def foo-bar-coercer
  (schema.coerce/coercer FooBarResult foo-bar-matcher))
```

Transformação/Coerção de fato:

```Clojure
(foo-bar-coercer valid-foor-bar-json-input)
=> {:test "FOO BAR"}

(foo-bar-coercer invalid-foor-bar-json-input)
=> #schema.utils.ErrorContainer{:error {"test-i" (not (= "FOO BAR" "Blue Pen"))}}
```

Quando executamos a chamada `(foo-bar-coercer valid-foor-bar-json-input)` temos o retorno esperado, onde o valor "FOO"
foi transformado em "FOO BAR" e o resultado atende ao schema especificado.

No segundo exemplo de chamada `(foo-bar-coercer invalid-foor-bar-json-input)`  o dado de entrada passou pelo step de
transformação, mas a saída não atendeu o schema esperado gerando um erro.

### Como isso é utilizado de forma mais prática e útil?

* Coerção de dados de entrada que estão no formato JSON onde todos os valores são strings ou números, conseguimos
  derivar tipos de dados mais específicos, como automaticamente converter datas em string para o tipo LocalDate interno
  do Java/Clojure, internalizar valores numéricos sem perder precisão.
    * Código fonte da lib com um exemplo de coerção de JSON para mapa
      Clojure: https://github.com/plumatic/schema/blob/master/src/cljc/schema/coerce.cljc


