# Arquitetura de Projeto

Este documento contém uma série de padrões de projetos que devem ser seguidos e respeitados. A arquitetura, aqui descrita, pode ser implementada em qualquer linguagem. A fim de demonstração, a linguagem adotada será `Javascript` com o uso de tipagem do `Typescript`.

> **Recomendação**: o toolkit `@piggly/ddd-toolkit` pode ser utilizado para auxiliar na implementação, uma vez que o objetivo dessa biblioteca é manter os componentes padrões organizados.

## Visão Geral

- **Camada de Domínio**: Entidades (`Entity`), Objetos de Valor (`ValueObject`), Agregados (`Entity`), Serviços de Domínio (`DomainService`) e Repositórios (`Repository`) por agregado;
  - Entidades e Agregados são conceitualmente diferentes, mas implementam a mesma classe `Entity`. A definição conceitual deles está na nomenclatura da classe-alvo. Exemplo: `OrderAggregateRoot extends Entity` e `OrderItemEntity extends Entity`.
- **CQRS**:
  - **Escrita**: Mensagens do tipo `WriteCommand` -> `ApplicationHandler` de escrita -> Repositório (`WritableRepository`) -> Eventos;
    - Retornam sempre uma Entidade, um Agregado ou uma Coleção de Entidades e Agregados;
    - Exemplo do pipeline com as nomenclaturas corretas: `CreateOrderWriteCommand` -> `CreateOrderApplicationHandler` -> `OrderWritableRepository`.
  - **Leitura**: Mensagens do tipo `ReadQuery` -> `ApplicationHandler` de leitura -> Repositório (`ReadableRepository`).
    - Retornam um `snapshot` (DTOs) de uma visualização dos dados;
    - Exemplo do pipeline com as nomenclaturas corretas: `OrderReadQuery` -> `OrderReadApplicationHandler` -> `OrderReadableRepository`.
- Mediador da Aplicação (`ApplicationMediator`):
  - Possuí um único ponto de envio de mensagem (`send(message)`, onde `message` é um `WriteCommand` ou `ReadQuery`) que aplica uma pipeline padronizada;
  - A pipeline é composta por **Middlewares** e o encaminhamento da mensagem para o `ApplicationHandler` correto. Este, por sua vez, será responsável por manipular os objetos da Camada de Domínio.

### Nomenclaturas

| **Conceito**                    | **Nome sugerido**                                            | **Função**                                          |
| ------------------------------- | ------------------------------------------------------------ | --------------------------------------------------- |
| Mensagem de **escrita**         | **`WriteCommand`** (`<Name>WriteCommand`)                    | Intenção de mudar estado.                           |
| Mensagem de **leitura**         | **`ReadQuery`** (`<Name>ReadQuery`)                          | Obter dados sem efeitos colaterais.                 |
| Função de Captura (**handler**) | **`<Name>ApplicationHandler`** (genérico); Um handler compatível com a mensagem enviada é acionado. | Orquestrar o caso de uso.                           |
| Mediador da Aplicação           | **`ApplicationMediator`**                                    | Aplica pipeline e roteia mensagem para os handlers. |
| Repositório (escrita)           | **`<Name>WriteRepository`**                                  | Hidrata/salva Agregado.                             |
| Repositório (leitura)           | **`<Name>ReadRepository`**                                   | Retorna **DTOs** (sem hidratar).                    |
| Entidade                        | **`<Name>Entity`** ou `<Name>AggregateRoot`                  | Possui identidade, invariantes.                     |
| VO                              | **`<Name>ValueObject`**                                      | Sem identidade, imutável.                           |
| Serviço de Domínio              | **`<Name>DomainService`**                                    | Regra/política que não “cabe” em uma entidade.      |
| Evento de Domínio               | **`<Name>DomainEvent`**                                      | Fato ocorrido no domínio.                           |
| Projeção                        | **`<Name>ViewRepository`**                                   | Leitura denormalizada, se houver.                   |
| DTO                             | **`<Name>DTO`**                                              | Dados para API/consumo externo.                     |

## Camada de Domínio

### Entidade (`Entity`)

| Rótulo         | Descrição                                                    |
| -------------- | ------------------------------------------------------------ |
| Camada         | Domínio                                                      |
| Pergunta-chave | A aplicação (ou usuário) precisa saber se este objeto é o mesmo de antes? |
| Comportamento  | Mutabilidade de atributos, permanência de identidade.        |

Entidades são objetos de domínio que possuem uma identidade contínua e única, independente da equidade dos atributos. Em outras palavras, **Cliente**, por exemplo, é uma Entidade que possuí uma identidade com um código único (id, geralmente), pois sempre representará aquele mesmo indivíduo. Mesmo que hajam clientes com mesmo nome, ainda serão clientes diferentes por conta da identidade única de cada um. Mesmo que o Cliente altere seu e-mail, ainda fazemos referência ao mesmo cliente de antes.

#### Características

- Possuem uma identidade única e exclusiva que não se altera ao longo do tempo: identificador auto incrementado, identificador aleatório (UUID), CPF de um indivíduo, CNPJ de uma empresa, SKU de um produto, etc;
- Possuem mutabilidade de atributos: podem mudar quaisquer valores internos (a depender das regras de cada Entidade), mas ainda permanecem sendo o mesmo objeto do ponto de vista do negócio. Exemplo: um pedido pode mudar de status e continua sendo o mesmo pedido;
- Duas Entidades com identidades diferentes não são e não devem ser consideradas iguais, mesmo que todos os atributos sejam semelhantes ou equivalentes. Exemplo: usuários com mesmo nome, ainda são usuários diferentes se possuírem identidades diferentes;
- Regras de negócio que modifiquem o estado da Entidade, devem fazer parte de um método exclusivo dentro da Entidade. À exemplo, a Entidade Produto pode ser modificada por uma oferta de desconto, a lógica de processamento deve estar em um método `ProductEntity.applyDiscount(value)`.

#### Boas Práticas

- Ao invés de utilizar o construtor padrão, utilizar um método estático dentro da própria Entidade (`Entity.create`) para atuar como `Factory` ou criar uma classe específica de montagem da entidade. Neste método (ou classe), você deve retornar um objeto `Result` ao invés de lançar uma exceção caso algum parâmetro não seja compatível;
- A Entidade não deve expor a mutabilidade de atributos complexos. Isso significa que supondo que a Entidade `User` tenha o atributo `allowedIps` com uma coleção de Objetos de Valor. A entidade não deve expor `allowedIps` de modo que a aplicação possa alterar diretamente o atributo `entity.allowedIps.add(ip)`, ao invés disso, deve-se expor mais métodos como `entity.allowIp(ip)`;
- Métodos de mutabilidade que possuam invariantes devem, preferencialmente, retornar um objeto `Result` com o erro relacionado. Isso garante que exceções sejam lançadas apenas por comportamentos inesperados e não regras do domínio;
- Garanta as invariantes do negócio com as Entidades. Isto é, faça com que as regras aplicáveis sejam sempre verdadeiras para garantir um estado válido (antes e depois de cada mudança). Se não forem verdadeiras, o estado da entidade foi quebrado,  portanto a operação deve ser bloqueada.
  - *Se eu puder salvar o estado mesmo com a regra falsa*, então **não** é invariante. Exemplo, “pedidos acima de R$ 200,00 ganham frete grátis” (caso a política não seja aplicável, o pedido ainda é válido);
  - *Se a regra precisa continuar verdadeira depois de qualquer operação*, então **é** invariante. Exemplo, “um pedido não pode ser confirmado sem itens adicionados” (o pedido não é válido se a política não for aplicável).

#### Implementação

> Os objetos abaixo estão disponíveis na biblioteca `@piggly/ddd-tookit`. Uma nova Entidade deve sempre estender `Entity`.

- `EntityID<Value = string>`: objeto usado por uma Entidade para guardar o valor da identidade. O valor dessa identidade pode ser uma `string`, um `number` ou um objeto complexo como `ObjectId`. Quando um novo `EntityID` é criado (sem uma carga inicial), ele é iniciado com um valor aleatório do mesmo tipo esperado para a Entidade. Esse valor, posteriormente pode ser reidratado pelo banco de dados;
  - `equals(entityId)`: testa a equivalência entre o identificador da Entidade, se verdadeiro,  portanto, significa que o identificador é o mesmo entre dois `EntityID` criados;
  - `isRandom()`: retorna se a identidade ainda está com valor aleatório (caso verdadeiro) ou se já é uma identidade fixa e persistente (caso falso);
  - `toNumber()`: força a conversão da identidade para o valor primitivo do tipo `number`;
  - `toString()`: força a conversão da identidade para o valor primitivo do tipo `string`;
  - `generateRandom()`: utilizado para implementar a geração de um valor aleatório para identidade. Ao criar um modelo de identidade, é recomendado substituir esse método. Ele deve ser chamado no construtor caso nenhum valor de identidade seja enviado como parâmetro.
  - Objetos mais literais podem ser utilizados, como: `StringEntityID`, `NumberEntityID` e `UUIDEntityID`.
  
- `Entity<Props, Id extends EntityID<any>>`: encapsula a Entidade (com a sua identidade) em um objeto. A Entidade pode ter qualquer conjunto de atributos (`Props`) e qualquer tipo de identidade compatível com `EntityID`;
  
  - O `_id` e `_props` são atributos do objeto que não devem estar acessíveis,  portanto devem são virtualmente protegidos (`protected`) para evitar acesso direto. Ainda, o `_id` deve ser um atributo com a característica de "apenas leitura". Para trocar a identidade de uma Entidade, deve-se,  portanto, cloná-la e alterar a identidade na construção;
  - `equals(entity)`: testa a equivalência entre a identidade de duas Entidade, se verdadeiro indica que os objetos testados são Entidades e possuem exatamente a mesma identidade (`EntityID`);
  - `generateId()`: método protegido chamado no construtor de uma Entidade quando a sua identidade não for enviada como parâmetro. Esse método deve ser substituído de acordo com o objeto de identidade que é usado e criado pela Entidade.
  - `emitter`: possuí uma propriedade protegida que é um objeto do tipo `EventEmmiter`. Este objeto deve ser usado para emitir eventos da Entidade (`Entity.emit(event, ...args)`) e ouvir (`Entity.on(event, (...args) => void)`). Também é possível desinscrever um handler (`Entity.off`) ou desinscrever todos (`Entity.dispose()`).
  
  - Possuí uma flag protegida (`_modified`) para identificar se a Entidade foi modificada ou não;
  - Possuí método `isModified()` para retornar o valor da flag;
  - Possuí métodos protegidos para gerenciamento de modificações. O `markAsModified()` para marcar a Entidade como modificada (irá emitir o evento `modified` e marcar a flag `_modified`); e, `markAsPersisted()` para desmarcar (irá emitir o evento `persisted` e desfazer a flag).
  
- `OptionalEntity<Entity extends IEntity<ID>, ID extends EntityID<any> = EntityID<any>>` é uma implementação que sempre carrega a identidade da Entidade, mas pode ou não carregar o objeto `Entity` hidratado. Esse objeto é útil para um “lazy loading” das Entidades. Você carrega a identidade da Entidade, mas não o carregamento da Entidade é opcional. Este carregamento pode ser feito no construtor ou utilizando o método `safeLoad(entity)`.
  - Possuí o método `load(entity)` que reidrata a Entidade, mas retorna um erro caso a identidade não seja a mesma. Para retornar um `Result`, deve-se utilizar `safeLoad(entity)`;
  - Possuí métodos para verificar se a Entidade está presente ou não (`isPresent()` e `isAbsent()`);
  - Possuí um `getter` para forçar o retorno da Entidade (`knowableEntity`). Isso garante que, caso a Entidade não esteja hidratada, uma exceção seja lançada para a aplicação;

##### Coleções

> Todos os objetos de Entidade podem ser armazenados em uma coleção aprimorada como `CollectionOfEntities`. Os métodos disponíveis para as coleções estão descritos abaixo.

- Propriedades:
  - `arrayOf`: retorna todos os itens como `array` de `OptionalEntity`;
  - `entries`: retorna `Iterator` de pares `[key, OptionalEntity]` da coleção;
  - `ids`: retorna `array` com todos as identidades das Entidades;
  - `keys`: retorna `array` com todas as chaves da coleção. As chaves são geradas a partir da conversão das identidades para `string` com o método `EntityID.toString()`;
  - `knowableEntities/entities`: retorna `array` apenas das Entidades que estão presentes (hidratadas);
  - `length`: retorna o número total de itens na coleção;
  - `values`: retorna `Iterator` com todos os valores `OptionalEntity`;
- Métodos
  - `add(item)`: adiciona uma Entidade à coleção (não recarrega se já existir, usar o método `sync` para esse comportamento);
  - `addMany(items)`: adiciona múltiplas Entidades de uma vez;
  - `appendRaw(item)`: anexa um `OptionalEntity` diretamente, sem verificação, substituindo sempre;
  - `appendManyRaw(items)`: anexa múltiplos `OptionalEntity` diretamente;
  - `sync(item)`: sincroniza uma Entidade, sempre substituindo se já existir;
  - `syncMany(items)`: sincroniza múltiplas Entidades de uma vez;
  - `find(id)`: busca uma Entidade pelo ID, retorna a Entidade ou `undefined`;
  - `forceFind(id)`: busca uma Entidade pelo ID, retorna a Entidade. O comportamento é forçado, ou seja, irá lançar uma exceção se não for encontrado. Deve ser usado quando você souber que a Entidade existe;
  - `get(id)`: obtém um `OptionalEntity` pelo ID;
  - `getKey(key)`: obtém um `OptionalEntity` pela chave;
  - `has(id)`: verifica se a coleção possui uma Entidade com o ID especificado;
  - `hasAll(ids)`: verifica se a coleção possui todas as Entidades pelos IDs;
  - `hasAllItems(items)`: verifica se a coleção possui todas as Entidades especificadas;
  - `hasAllKeys(keys)`: verifica se a coleção possui todas as chaves especificadas;
  - `hasAny(ids)`: verifica se a coleção possui alguma das Entidades pelos IDs;
  - `hasAnyItems(items)`: verifica se a coleção possui alguma das Entidades especificadas;
  - `hasAnyKeys(keys)`: verifica se a coleção possui alguma das chaves especificadas;
  - `hasItem(item)`: verifica se a coleção possui a Entidade especificada;
  - `hasKey(key)`: verifica se a coleção possui a chave especificada;
  - `itemAvailableFor(id)`: verifica se a Entidade está disponível (presente) para o ID;
  - `remove(id)`: remove uma Entidade pelo ID;
  - `removeItem(item)`: remove uma Entidade específica;
  - `removeKey(key)`: remove uma Entidade pela chave;
  - `reload(item)`: recarrega uma Entidade que já existe na coleção, se não existir lança uma exceção;
  - `reloadMany(items)`: recarrega múltiplas Entidades existentes na coleção, se uma das Entidades não existir lança uma exceção;
  - `clone()`: cria uma cópia da coleção.

##### Exemplo

```ts
import { Entity, NumberEntityId, InvalidPayloadSchemaError, Result } from "@piggly/ddd-toolkit";
import { z } from 'zod';

export type ConcreteEntityProps = {
    name: string;
    value: string;
};


class ConcreteEntity extends Entity<ConcreteEntityProps, NumberEntityId> {
	public get name() {
		return this._props.name;
	}

	public get value() {
		return this._props.value;
	}

	public updateName(name: string) {
		this._props.name = name;
        this.markAsModified();
	}
    
    public clone(id?: NumberEntityId): ConcreteEntity {
        return new ConcreteEntity({
            name: this._props.name,
            value: this._props.value
        }, id ?? this._id)
    }
    
    public static create(props: any) {
        const parsed = z.object({
            name: z.string().min(3),
            value: z.string().min(3)
        }).safeParse(props);
        
		if (parsed.success === false) {
			return Result.fail(
				new InvalidPayloadSchemaError(
					'InvalidConcreteEntityError',
					'Expected parameters missmatch.',
					parsed.error.issues,
				),
			);
		}
        
        return Result.ok(new ConcreteEntity(parsed));
    }

	protected generateId() {
		return new NumberEntityId();
	}
}

export default ConcreteEntity;
```

#### Agregados

### Objeto de Valor (`ValueObject`)

| Rótulo         | Descrição                                                    |
| -------------- | ------------------------------------------------------------ |
| Camada         | Domínio                                                      |
| Pergunta-chave | Corresponde a um atributo que pode ser descartável, substituível e imutável? |
| Comportamento  | Imutabilidade de valores. O objeto sempre será o mesmo e será equivalente a outro com os mesmos atributos. |

Objetos de Valor são objetos de domínio que **não** possuem uma identidade. Eles são descartáveis e equivalentes quando os atributos são mesmos para dois objetos. Em outras palavras, **Cor**, por exemplo, é um objeto de valor que pode ter um hexadecimal e um nome. Independente de quantas instâncias de `{ name: "White", hex: "#fff" }` sejam criadas, todas elas serão as mesmas uma vez que todos os seus atributos são iguais.

#### Características

- Objetos de Valor sempre são imutáveis. Se um atributo do Objeto de Valor precisar mudar, então, devemos criar um novo Objeto de Valor com os novos atributos. A imutabilidade evita efeitos colaterais e problemas de identidade;
- Comparação por valor intrínseco: a igualdade entre dois Objetos de Valor é determinada sempre comparando seus atributos. Se todos eles forem iguais, então os objetos são os mesmos (mesmo que sejam instâncias diferentes na memória);
- São descartáveis ou substituíveis: pense nos Objetos de Valor como tipos primitivos aprimorados do domínio. Assim como você pode alterar um número (que permanece como um número), você pode alterar um Objeto de Valor por outro do mesmo tipo;
- Auto-validação: Objetos de Valor não guardam regras de negócios, entretanto eles costumam garantir que os seus atributos passem por uma validação intrínseca. Por exemplo, um Objeto de Valor “E-mail”, deve validar se o e-mail é correto durante a construção do mesmo;
- A mutabilidade dos Objetos de Valor geralmente é controlada pela Entidade a qual ele pertence. Quando não for o caso, a Entidade já deve exigir um Objeto de Valor do tipo certo. Exemplo: é possível ter o método `changeColor(name, hex)` onde a Entidade é responsável por criar o novo Objeto de Valor ou o método `changeColor(vo)` que já recebe o Objeto de Valor “Cor” já criado.

#### Boas Práticas

- Ao invés de utilizar o construtor padrão, utilizar um método estático dentro do próprio Objeto de Valor (`ValueObject.create`) para atuar como `Factory` ou criar uma classe específica de montagem do Objeto de Valor. Neste método (ou classe), você deve retornar um objeto `Result` ao invés de lançar uma exceção caso algum parâmetro não seja compatível;
- Objetos de Valor podem possuir métodos de mutabilidade, porém esses métodos devem retornar um novo Objeto de Valor para ser reatribuído. Por exemplo, `Money.add(money)` irá retornar um novo Objeto de Valor com o valor monetário atualizado. Preferencialmente, ao aplicar a mutação, retornar um objeto `Result` com o erro relacionado. Isso garante que exceções sejam lançadas apenas por comportamentos inesperados e não regras do domínio;
- Garanta idempotência onde for apropriado, isto é: se uma ação não altera o estado, não falhe e não execute.

#### Implementação

> Os objetos abaixo estão disponíveis na biblioteca `@piggly/ddd-tookit`. Um Objeto de Valor deve sempre estender `ValueObject`.

`ValueObject<Props extends Record<string, any> = Record<string, any>> implements IValueObject<Props>` é uma classe que recebe propriedade devidamente tipadas e as congela com `Object.freeze` impedindo a mutabilidade. Outros métodos também estão disponíveis:

- `equals(vo)`: retorna se os Objetos de Valor são idênticos;
- `hash()`: retorna o hash `sha256` do Objeto de Valor;

##### Coleções

> Todos os Objetos de Valor podem ser armazenados em uma coleção aprimorada como `CollectionOfValueObjects`. Os métodos disponíveis para as coleções são similares à coleção de Entidades.

##### Exemplo

```ts
import { ValueObject, Result, DomainError, InvalidPayloadSchemaError } from "@piggly/ddd-toolkit";
import { z } from 'zod';

export type PositiveNumberValueObjectProps = {
    value: number;
}

class PositiveNumberValueObject extends ValueObject<PositiveNumberValueObjectProps> {
	public constructor(num: number) {
		super({ value: num });
	}

	public get value(): string {
		return this.props.value;
	}
    
    public add(num: PositiveNumberValueObject): PositiveNumberValueObject {
        if (num.value === 0) {
            return Result.ok(this);
        }
        
        return new PositiveNumberValueObject(num.value + this.value);
    }
    
    public static create(num: number): Result<PositiveNumberValueObject, DomainError> {
        const parsed = z.coerse.number().int().positive();
        
		if (parsed.success === false) {
			return Result.fail(
				new InvalidPayloadSchemaError(
					'InvalidPositiveNumberValueObjectError',
					'Expected parameters missmatch.',
					parsed.error.issues,
				),
			);
		}
        
        return Result.ok(new PositiveNumberValueObject(parsed));
    }
}

export default PositiveNumberValueObject;
```

### Repositórios (`Repository`)

| Rótulo         | Descrição                                                    |
| -------------- | ------------------------------------------------------------ |
| Camada         | Domínio / Infraestrutura (contratos no Domínio; implementações na Infraestrutura) |
| Pergunta-chave | **Preciso hidratar um Agregado completo ou só ler dados para exibição?** |
| Comportamento  | `WritableRepository`: hidrata/salva Agregados. `ReadableRepository`: retorna **`DTOs`**. |

Repositórios abstraem a persistência e leitura de dados. Na implementação desta arquitetura, em conformidade com o padrão **CQRS**, os repositórios, separam-se em:

- **`<Name>WritableRepository`**: operações de **escrita** e carregamento de Agregados para aplicar regras/invariantes. Por exemplo, `OrderWritableRepository` é responsável por escrever todo o agregado de pedidos em um banco de dados;
  - Repositórios de escrita podem ter implementações de leitura, por exemplo: `OrderWritableRepository.getById`. Neste caso, ao chama o método `getById` todo o agregado do pedido será populado. Isso será essencial para fazer novas operações de escrita sobre ele. Desse modo, implementações de leitura só devem ser implementadas aqui quando forem essenciais para operações de escrita posteriores.
- **`<Name>ReadableRepository`**: operações de **leitura** que retornam `DTOs` (sem hidratar agregados). Os métodos de leitura podem ter **paginação/filtros** eficientes. Por exemplo, `OrderReadableRepository` é responsável por ler os dados do agregado, sem hidratar uma entidade;
  - Um exemplo eficiente de uso é quando a aplicação deseja listar os pedidos com apenas algumas informações, como: ID, nome do cliente, data do pedido e status. Não é necessário carregar a entidade completa e o esquema de dados retornado deve ser compatível com o que é esperado para visualização. Geralmente esse dado é encapsulado em um `DTO`.

#### Características

- A implementação do Repositório fica na Camada de Domínio da aplicação. Por outro lado, as implementações de banco de dados (sejam quais forem) vivem na Camada de Infraestrutura da Aplicação. Sendo assim, o driver do banco de dados será injetado como dependência do Repositório;

- O Repositório de Escrita sempre irá carregar o agregado completo (de modo a respeitar as invariantes) e também salvará esse agregado como uma unidade;

- O Repositório de Leitura nunca deve retornar nem entidades e muito menos agregados. Sempre irá retornar `DTOs` que podem conter dados parciais (ou completos) de um `snapshot` de uma entidade;

- Ambos tipos de Repositórios tem a finalidade de isolar a tecnologia (driver de implementação do banco de dados) da Camada de Domínio através da injeção de dependências;
- Não valida regras de negócio. O Repositório deve receber parâmetros já padronizados e executar as operações. Desse modo, o Repositório faz defesa mínima (`null/undefined`, ou parâmetros que afetam a estabilidade query);
- A implementação de infraestrutura que é responsável por lançar erros quando algo der errado. O repositório deve propagar esses erros. Mas ao resolver a query, por outro lado, o repositório deve retornar `undefined` quando não conseguir obter uma resultado ou `false` para casos em que não seja necessário retornar dados explícitos.

#### Boas Práticas

- Um Repositório não deve “vazar” detalhes da infraestrutura utilizada (queries, drivers, conexões, etc) para o domínio;
- Nos métodos de leitura, sempre exponha métodos orientados a casos de uso (por exemplo, `getById(EntityID)`). Evite utilizar CRUD genérico ao desenhar os métodos (por exemplo, `get`, `create`, `update`, etc). Seja explícito;
- Dentro de um Repositório de Escrita, sempre coloque as hidratações necessárias para os métodos de estrita para garantir que as invariantes das Entidades ou Agregados manipulados sejam respeitadas;
- Nunca retorne Entidades em um Repositório de Leitura;
  - É preferencial que os parâmetros dos Repositórios de Leitura sejam um Objeto de Valor (`ValueObject`) específico, montado na Camada da Aplicação, antes de chamar o Repositório;
  - Os Objetos de Valor utilizados devem ser infra-agnósticos, eles não devem saber nada que seja relacionado ao banco de dados. Por exemplo, o objeto `PaginationValueObject` não sabe nada sobre `offset`/`limit`, ele sabe sobre `page`/`size`. Além disso, esse objeto não faz a conversão dos valores, quem faz isso é o Repositório.
- Sempre retorne Entidades em um Repositório de Escrita.
  - É preferencial que os parâmetros dos Repositórios de Escrita sejam uma Entidade (`Entity`) ou uma Identidade da Entidade (`EntityID`).

#### Implementação

> A biblioteca `@piggly/ddd-tookit` não força uma implementação para repositórios. Esteja livre para criar. Você pode seguir as regras abaixo para se organizar melhor.

- No construtor de um Repositório, injete todas as dependências de infraestrutura;
- Crie métodos independentes e isolados. Deixe a Camada da Aplicação chamar as operações necessárias, como: carregar o Agregado, fazer as alterações e então salvar o Agregado.

### Serviços de Domínio (`DomainService`)

| Rótulo         | Descrição                                                    |
| -------------- | ------------------------------------------------------------ |
| Camada         | Domínio                                                      |
| Pergunta-chave | A regra depende de múltiplos Agregados ou não “cabe” naturalmente em um só? |
| Comportamento  | Políticas de negócio coesas e reutilizáveis, preferencialmente **puras** (sem I/O). |

Serviços de Domínio encapsulam regras e políticas de negócio que:

- Dependem de mais de um agregado (ou não pertencem claramente a um único);
- Precisam ser reutilizadas por várias Entidades ou Handlers;
- Podem ser puras (sem I/O). Quando dependerem de I/O, deve-se injetar portas (interfaces) do domínio.

#### Características

- Não possuem estado, isso significa que são um conjunto de funções (relacionadas) puras. Um dado entra, o resultado saí;
- Focam em política sempre, não em orquestração. A orquestração é uma responsabilidade da Camada da Aplicação;
- São reutilizáveis, ou seja, podem ser chamados por Entidades e Handlers sem duplicar lógica.

#### Boas Práticas

- Preferir pureza na implementação, portanto, não deve produzir “side effects”;
  - Não escreva arquivos ou banco de dados;
  - Não chame HTTP, filas ou serviços externos à aplicação;
  - Não publique eventos;
  - Não produza logs relevantes;
  - Não leia variáveis externas (exemplo: `process.env`, `Date.now()`);
  - Não gere valores importados de dependências externas do projeto, ao invés disso crie uma interface.
- Não utilize operações de entrada e saída (I/O) dentro do serviço;
  - Operações de entrada e saída incluí: leitura no banco de dados, leitura via protocolos de rede, leitura de arquivos, aleatoriedade, clock, etc;
  - Caso a operação seja indispensável, por exemplo, criptografar um valor de entrada para produzir uma saída. Então, as operações de criptografia devem ser ser injetadas como dependência (via interface). Neste caso, criar uma interface `ICryptoService` que tenha os métodos `hash()`, `encrypt()` e `decrypt()`, por exemplo.
- Deve ser focado em uma única atividade. Exemplo, “política de cálculo de desconto” é diferente de “política de cálculo de impostos”, ainda que ambas calculem mudanças no valor de um pedido;
- Os parâmetros de um método implementável são livres para receber um Objeto de Valor ou uma Entidade (não pode sofrer mutação no serviço). O objetivo é apenas ler os dados e retornar os resultados de uma operação. Se você aplica uma política de cálculo de desconto, você pode ler a Entidade de Pedido, mas deverá retornar o cálculo independente. A Camada de Aplicação que decide o que fazer com o dado retornado;
- O nome do serviço deve respeitar a Linguagem Ubíqua, sempre se referindo a política implementada;
- Não deve depender de infraestrutura (serviços externos), apenas interfaces do domínio. Todos os adaptadores ficam na infra.

#### Regras de Negócio: no Serviço ou na Entidade?

Para decidir se uma regra de negócio deve fazer parte de um Serviço de Domínio ou de uma Entidade. Você deve percorrer o checklist abaixo e garantir o posicionamento correto das regras:

1. A regra aplicável depende de mais de um agregado (ou entidades distintas)? Se aplicável, então Serviço de Domínio;
2. A regra depende de infraestrutura? Se aplicável, então Serviço de Domínio;
3. A regra é responsável por uma invariável interna do Agregado? Se aplicável, então na raiz do Agregado ou no Objeto de Valor;
4. É um cálculo puro sobre os valores sem identidade? Se aplicável, no Objeto de Valor ou uma função pura;
5. É uma orquestração de passos (carregar entidades, chamar domínios, persistir, etc)? Se aplicável, então Serviço de Aplicação;
6. É uma política “plugável” ou frequentemente mutável? Se aplicável, então Serviço de Domínio.

#### Implementação

> A biblioteca `@piggly/ddd-tookit` não força uma implementação para serviços de domínio, embora você possa estender `DomainService` apenas para fins de classificação. Esteja livre para criar. Você pode seguir as regras abaixo para se organizar melhor.

- No construtor de um Serviço de Domínio, injete todas as dependências externas como interfaces do Domínio ou operações de I/O.

### Eventos de Domínio (`DomainEvent`)

| Rótulo         | Descrição                                                    |
| -------------- | ------------------------------------------------------------ |
| Camada         | Domínio (emissão); Aplicação (assinatura/processamento).     |
| Pergunta-chave | Um fato relevante do domínio ocorreu que outros componentes devem saber? |
| Comportamento  | Fato imutável publicado quando uma operação válida altera o estado do domínio. |

Eventos de Domínio narram **fatos** (por exemplo, `OrderPaidDomainEvent`). São emitidos pelo Agregado quando uma mudança válida ocorre. A camada de aplicação os **ouve** para disparar ações (por exemplo, notificar o cliente), mantendo baixo acoplamento.

> Não confundir com Eventos Locais de uma Entidade. Os eventos Locais de uma entidade servem para propagar mudanças de estado entre partes da Entidade. Por outro lados, Eventos de Domínio são um fato concreto que pode ser capturado pela aplicação e direcionado para ações específicas.

> É recomendado usar a biblioteca [@piggly/event-bus](https://www.npmjs.com/package/@piggly/event-bus). A biblioteca está preparada para receber e disparar eventos de qualquer lugar da aplicação.

#### Características

- Os Eventos de Domínio devem ser imutáveis e descritivos, além disso o nome deve estar sempre no passado;
- Os Agregados são responsáveis por emitir os eventos, enquanto os Handler e Gerenciadores de Processos devem reagir;
- Mantem baixo acoplamento, uma vez que os produtores não conhecem e não se importam com os consumidores;
- A entrega é no mínimo “best effort”; para robustez é recomendado usar uma **Outbox**.

#### Boas práticas

- Nomeie o evento com o fato concreto (ubíquo);
- Os parâmetros do evento devem ser montados preferencialmente estendendo `EventPayload` da biblioteca `@piggly/event-bus`;
- Emitir na raiz do Agregado quando o fato acontece;
- Não colocar regras de negócio no handler do evento, use-o apenas para orquestrar (enviar para uma Queue, por exemplo);
- Utilizar Outbox se precisar garantir publicação após commit.

### Considerações Gerais

Ainda existem alguns conceitos de domain-driven design que devem ser implementados no fluxo da aplicação:

- **Fábricas (`Factory`)**: Fábricas são úteis para criar Agregados complexos. Em vez de ter `new OrderAggregateRoot(...)` exposto, poderíamos ter um método estático `OrderAggregateRoot.create(args)` ou uma classe `CreateOrderFactory` que monta um pedido válido. A Fábrica encapsula o processo de montagem respeitando invariantes;

- **Contextos Delimitados (Bounded Contexts)**: DDD estratégico reconhece que um grande sistema pode ser dividido em múltiplos subdomínios, cada um com seu modelo. Nosso exemplo é focado em um contexto (vendas). Em um sistema real, o **Contexto de Vendas** (pedidos, produtos, pagamentos) poderia ser distinto do **Contexto de Catálogo** (gerência de produtos, categorias) ou de **Usuários**. Isso implica que as mesmas palavras podem ter significados diferentes em contextos diferentes. Bounded contexts são normalmente implementados como módulos ou microsserviços separados;

- **Linguagem Ubíqua**: Ao implementar, sempre use nomes em inglês e contextuais. Isso é deliberado para espelhar a linguagem do negócio. É necessário manter a consistência com os termos de domínio usados pelos stakeholders e aplicáveis ao contexto. O importante é que todos usem os mesmos termos para evitar confusão;

- **Teste de Unidade e Design**: DDD encoraja desenho focado em domínio que costuma resultar em código mais testável. Entidades e Serviços de domínio não dependem de infraestrutura, então podemos instanciá-los em testes simples. Por exemplo, testar `OrderAggregateRoot.addItem` invariantes, ou `DiscountPoliciesDomainService.calculate` sem precisar de banco ou HTTP. Isso leva a maior confiança no core do sistema.

- **Camada de Infraestrutura**: Ainda que não seja foco, lembre-se que eventualmente terá implementação de Repositórios (ex: usando Sequelize, TypeORM, mongoose, etc.), implementação real de envio de E-mail, logs, etc. Essas partes devem ficar de fora do domínio. Uma prática comum é usar **interfaces** ou classes abstratas na Camada de Domínio para representar, por exemplo, um `IEmailService` genérico, e na infra ter `EmailService` com envio via SMTP ou API. Depois, injetar essa implementação na Camada de Aplicação. Assim, se um dia mudar a forma de envio de e-mail, nada no domínio muda.

## Todas as Camadas

### Tratamento de Erros (`Result`)

| Rótulo         | Descrição                                                    |
| -------------- | ------------------------------------------------------------ |
| Camada         | **Aplicação e Domínio** (propagação de sucesso/falha como **valor**); conversão final na borda de saída da aplicação (HTTP, CLI, etc). |
| Pergunta-chave | **Quero sinalizar uma falha de regra/validação que não é “intratável”?** |
| Comportamento  | Encapsula **sucesso** (`ok(data)`) ou **falha** (`fail(error)`) e permite **composição** segura (`chain/map`). |

O objeto `Result<Data, Error extends DomainError>` evita controlar fluxo com `throw` para casos esperados (como validações, regras de negócio e procedimentos tratáveis pelo usuário). É possível compor os passos que podem falhar e somente no limite da borda (por exemplo, na resposta da requisição HTTP) o resultado é convertido para status/erro.

#### Características

- O sucesso ou a falha são convertidos para valores no mesmo objeto. Com a propriedade `Result.isSuccess` é possível verificar se o resultado foi bem sucedido ou, ainda, com `Result.isFailure` ao contrário;
- Se bem sucedido, o objeto terá populado a propriedade `data` com o valor esperado, do contrário a propriedade `error` será populada com um objeto de erro que estende `DomainError`;
- Com o objeto `Result` ainda é possível utilizar composição para evitar o uso excessivo de `if`:
  - `.chain(fn)` encadeia passos que retornam `Result` (sejam a partir de funções síncronas ou assíncronas). Se falhar, propaga o erro para todas as demais funções encadeadas;
  - `.map(fn)` transforma apenas o `data` do resultado anterior, retornando um novo objeto `Result` para continuar a composição;
  - `.mapError(fn)` transforma apenas o `error` do resultado anterior, retornando um novo objeto `Result` para continuar a composição;
  - `.tap(fn)` executa efeito colateral, com base no resultado anterior, sem alterar o `Result`. O efeito colateral só é executado se houver sucesso, do contrário é ignorado.
- Na Camada de Domínio, Entidades, Objetos de Valor e Serviços de Domínio, podem retornar um `Result.fail` para regras esperadas (invariantes e validação);
  - `BusinessRuleViolationError` pode ser usado com extensão de erros relacionados a invariantes;
  - `InvalidPayloadSchemaError` pode ser usado como extensão de erros relacionados a validação.
- Na Camada da Aplicação, Mediador, Handlers, Comandos, Queries e Serviços da Aplicação, podem retornar um `Result.fail` para regras esperadas (invariantes e validação);
  - `BusinessRuleViolationError` pode ser usado com extensão de erros relacionados a invariantes do domínio;
  - `InvalidPayloadSchemaError` pode ser usado como extensão de erros relacionados a validação;
  - `ApplicationError` pode ser usado como extensão de erros gerais relacionados a aplicação.
- Na Camada de Infraestrutura, evitar usar o `Result`, neste caso, é preferível lançar exceções e, opcionalmente, estender o objeto `RuntimeError`. Na última ponta da aplicação, pode ser convertido para um erro de apresentação para o usuário, usando o erro verdadeiro apenas para logging e monitoramento.

#### Boas Práticas

- **Onde usar `Result`**:

  - Na Camada de Domínio para invariantes e regras esperadas.

  - `ApplicationHandler` para **orquestrar** vários passos com `.chain`.

- **Onde NÃO abusar**:
  - Não esconda bugs/erros de infraestrutura. Deixe que esses serviços lancem exceções, e mapeie na aplicação (`try/catch`) se o erro for tratável, do contrário prefira `graceful shutdown`.

- **Tipagem forte**:
  - Padronize um **catálogo de `DomainError`** (por exemplo, `ValidationError`, `NotFoundError`, `RuleViolationError`, `UnauthorizedError`, `InfraError`). As bibliotecas `@piggly` e `@pgly` já possuem diversos templates de erros mais usados. Sempre consulte antes de criar um novo erro.

- **Borda HTTP/CLI**:
  - Converta `DomainError` para um erro de exibição. Você pode usar o método `toJSON()` para exibir as propriedades do erro e o parâmetro `status` para retornar o status HTTP relacionado.

- **Efeito colateral**:
  - Use `.tap` **somente** na **camada de aplicação** (logs/observabilidade). Evite em domínio.

#### Regras de bolso (rápidas)

- **Regras esperadas** (validação/invariante) → `Result.fail(new DomainError(...))`;
- **Falhas inesperadas** (I/O, bug) → `throw`; a Aplicação captura e mapeia para um erro genérico como `RequestServerError`;
- Componha pipelines com `.chain` (`async/sync`), **sem** `try/catch` a cada passo;
- Converta para HTTP uma única vez na borda. Ao construir API’s o uso de `@piggly/fastify-chassis` é recomendado;
- **Use `.tap`** para logs/observabilidade na Aplicação, não no Domínio.
