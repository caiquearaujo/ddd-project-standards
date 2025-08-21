# Arquitetura de Projeto

Este documento contém uma série de padrões de projetos que devem ser seguidos e respeitados. A arquitetura, aqui descrita, pode ser implementada em qualquer linguagem. A fim de demonstração, a linguagem adotada será `Javascript` com o uso de tipagem do `Typescript`.

> ⚠️ **Recomendação**: o toolkit `@piggly/ddd-toolkit` pode ser utilizado para auxiliar na implementação, uma vez que o objetivo dessa biblioteca é manter os componentes padrões organizados.

## Visão Geral

- **Camada do Domínio**: ENTIDADES (`Entity`), OBJETOS DE VALOR (`ValueObject`), AGREGADOS (`Entity`), SERVIÇOS DE DOMÍNIO (`DomainService`) e REPOSITÓRIOS (`Repository`) por AGREGADO;
  - ENTIDADES e AGREGADOS são conceitualmente diferentes, mas implementam a mesma classe `Entity`. A definição conceitual deles está na nomenclatura da classe-alvo. Exemplo: `OrderAggregateRoot extends Entity` e `OrderItemEntity extends Entity`.
- **CQRS**:
  - **Escrita**: Mensagens do tipo `WriteCommand` -> `ApplicationHandler` de escrita -> REPOSITÓRIO (`WritableRepository`) -> Eventos;
    - Retornam sempre uma ENTIDADE, um AGREGADO ou uma Coleção de ENTIDADES e AGREGADOS;
    - Exemplo do pipeline com as nomenclaturas corretas: `CreateOrderWriteCommand` -> `CreateOrderApplicationHandler` -> `OrderWritableRepository`.
  - **Leitura**: Mensagens do tipo `ReadQuery` -> `ApplicationHandler` de leitura -> REPOSITÓRIO (`ReadableRepository`).
    - Retornam um `snapshot` (DTOs) de uma visualização dos dados;
    - Exemplo do pipeline com as nomenclaturas corretas: `OrderReadQuery` -> `OrderReadApplicationHandler` -> `OrderReadableRepository`.
- **Mediador da Aplicação** (`ApplicationMediator`):
  - Possuí um único ponto de envio de mensagem (`send(message)`, onde `message` é um `WriteCommand` ou `ReadQuery`) que aplica uma pipeline padronizada;
  - A pipeline é composta por **Middlewares** e o encaminhamento da mensagem para o `ApplicationHandler` correto. Este, por sua vez, será responsável por manipular os objetos da  Camada do Domínio.

### Nomenclaturas

| **Conceito**                    | **Nome sugerido**                                            | **Função**                                          |
| ------------------------------- | ------------------------------------------------------------ | --------------------------------------------------- |
| Mensagem de **escrita**         | **`WriteCommand`** (`<Name>WriteCommand`)                    | Intenção de mudar estado.                           |
| Mensagem de **leitura**         | **`ReadQuery`** (`<Name>ReadQuery`)                          | Obter dados sem efeitos colaterais.                 |
| Função de Captura (**handler**) | **`<Name>ApplicationHandler`** (genérico); Um handler compatível com a mensagem enviada é acionado. | Orquestrar o caso de uso.                           |
| Mediador da Aplicação           | **`ApplicationMediator`**                                    | Aplica pipeline e roteia mensagem para os handlers. |
| REPOSITÓRIO (escrita)           | **`<Name>WriteRepository`**                                  | Hidrata/salva AGREGADO.                             |
| REPOSITÓRIO (leitura)           | **`<Name>ReadRepository`**                                   | Retorna **DTOs** (sem hidratar).                    |
| ENTIDADE                        | **`<Name>Entity`** ou `<Name>AggregateRoot`                  | Possui identidade, invariantes.                     |
| VO                              | **`<Name>ValueObject`**                                      | Sem identidade, imutável.                           |
| SERVIÇO DE DOMÍNIO              | **`<Name>DomainService`**                                    | Regra/política que não “cabe” em uma entidade.      |
| Evento de Domínio               | **`<Name>DomainEvent`**                                      | Fato ocorrido no domínio.                           |
| Projeção                        | **`<Name>ViewRepository`**                                   | Leitura denormalizada, se houver.                   |
| DTO                             | **`<Name>DTO`**                                              | Dados para API/consumo externo.                     |

### Camadas

- **Camada de Apresentação (interface do usuário)**: responsável por mostrar informações ao usuário e interpretar os comandos enviador por ele. Esse agente externo pode, às vezes, ser outro sistema de computador ao invés de um usuário humano;
- **Camada da Aplicação**: define as funções que o software deve executar e direciona objetos expressivos do domínio para resolver os problemas. Não contém regras ou conhecimento do negócio, apenas coordena as tarefas e delega trabalhos para conjuntos de objetos e o domínio na camada logo abaixo;
- **Camada do Domínio**: Responsável por representar conceitos do negócio, informações sobre as situações dos negócios e aplicação das regras de negócio. Esta camada é o verdadeiro coração do software;
- **Camada da Infraestrutura**: Fornece recursos técnicos genéricos (interfaces) que suportam as camadas mais altas: envio de mensagens, persistência de dados, etc.

## Camada do Domínio

### ENTIDADE (`Entity`)

| Rótulo         | Descrição                                                    |
| -------------- | ------------------------------------------------------------ |
| Camada         | Domínio                                                      |
| Pergunta-chave | A aplicação (ou usuário) precisa saber se este objeto é o mesmo de antes? |
| Comportamento  | Mutabilidade de atributos, permanência de identidade.        |

ENTIDADES são objetos de domínio que possuem uma identidade contínua e única, independente da equidade dos atributos. Em outras palavras, **Cliente**, por exemplo, é uma ENTIDADE que possuí uma identidade com um código único (id, geralmente), pois sempre representará aquele mesmo indivíduo. Mesmo que hajam clientes com mesmo nome, ainda serão clientes diferentes por conta da identidade única de cada um. Mesmo que o Cliente altere seu e-mail, ainda fazemos referência ao mesmo cliente de antes.

#### Características

- Possuem uma identidade única e exclusiva que não se altera ao longo do tempo: identificador auto incrementado, identificador aleatório (UUID), CPF de um indivíduo, CNPJ de uma empresa, SKU de um produto, etc;
- Possuem mutabilidade de atributos: podem mudar quaisquer valores internos (a depender das regras de cada ENTIDADE), mas ainda permanecem sendo o mesmo objeto do ponto de vista do negócio. Exemplo: um pedido pode mudar de status e continua sendo o mesmo pedido;
- Duas ENTIDADES com identidades diferentes não são e não devem ser consideradas iguais, mesmo que todos os atributos sejam semelhantes ou equivalentes. Exemplo: usuários com mesmo nome, ainda são usuários diferentes se possuírem identidades diferentes;
- Regras de negócio que modifiquem o estado da ENTIDADE, devem fazer parte de um método exclusivo dentro da ENTIDADE. À exemplo, a ENTIDADE Produto pode ser modificada por uma oferta de desconto, a lógica de processamento deve estar em um método `ProductEntity.applyDiscount(value)`.

#### Boas Práticas

- Ao invés de utilizar o construtor padrão, utilizar um método estático dentro da própria ENTIDADE (`Entity.create`) para atuar como `Factory` ou criar uma classe específica de montagem da entidade. Neste método (ou classe), você deve retornar um objeto `Result` ao invés de lançar uma exceção caso algum parâmetro não seja compatível;
- A ENTIDADE não deve expor a mutabilidade de atributos complexos. Isso significa que supondo que a ENTIDADE `User` tenha o atributo `allowedIps` com uma coleção de OBJETOS DE VALOR. A entidade não deve expor `allowedIps` de modo que a aplicação possa alterar diretamente o atributo `entity.allowedIps.add(ip)`, ao invés disso, deve-se expor mais métodos como `entity.allowIp(ip)`;
- Métodos de mutabilidade que possuam invariantes devem, preferencialmente, retornar um objeto `Result` com o erro relacionado. Isso garante que exceções sejam lançadas apenas por comportamentos inesperados e não regras do domínio;
- Garanta as invariantes do negócio com as ENTIDADES. Isto é, faça com que as regras aplicáveis sejam sempre verdadeiras para garantir um estado válido (antes e depois de cada mudança). Se não forem verdadeiras, o estado da entidade foi quebrado,  portanto a operação deve ser bloqueada.
  - *Se eu puder salvar o estado mesmo com a regra falsa*, então **não** é invariante. Exemplo, “pedidos acima de R$ 200,00 ganham frete grátis” (caso a política não seja aplicável, o pedido ainda é válido);
  - *Se a regra precisa continuar verdadeira depois de qualquer operação*, então **é** invariante. Exemplo, “um pedido não pode ser confirmado sem itens adicionados” (o pedido não é válido se a política não for aplicável).

#### Implementação

> Os objetos abaixo estão disponíveis na biblioteca `@piggly/ddd-tookit`. Uma nova ENTIDADE deve sempre estender `Entity`.

- `EntityID<Value = string>`: objeto usado por uma ENTIDADE para guardar o valor da identidade. O valor dessa identidade pode ser uma `string`, um `number` ou um objeto complexo como `ObjectId`. Quando um novo `EntityID` é criado (sem uma carga inicial), ele é iniciado com um valor aleatório do mesmo tipo esperado para a ENTIDADE. Esse valor, posteriormente pode ser reidratado pelo banco de dados;
  - `equals(entityId)`: testa a equivalência entre o identificador da ENTIDADE, se verdadeiro,  portanto, significa que o identificador é o mesmo entre dois `EntityID` criados;
  - `isRandom()`: retorna se a identidade ainda está com valor aleatório (caso verdadeiro) ou se já é uma identidade fixa e persistente (caso falso);
  - `toNumber()`: força a conversão da identidade para o valor primitivo do tipo `number`;
  - `toString()`: força a conversão da identidade para o valor primitivo do tipo `string`;
  - `generateRandom()`: utilizado para implementar a geração de um valor aleatório para identidade. Ao criar um modelo de identidade, é recomendado substituir esse método. Ele deve ser chamado no construtor caso nenhum valor de identidade seja enviado como parâmetro.
  - Objetos mais literais podem ser utilizados, como: `StringEntityID`, `NumberEntityID` e `UUIDEntityID`.
  
- `Entity<Props, Id extends EntityID<any>>`: encapsula a ENTIDADE (com a sua identidade) em um objeto. A ENTIDADE pode ter qualquer conjunto de atributos (`Props`) e qualquer tipo de identidade compatível com `EntityID`;
  
  - O `_id` e `_props` são atributos do objeto que não devem estar acessíveis,  portanto devem são virtualmente protegidos (`protected`) para evitar acesso direto. Ainda, o `_id` deve ser um atributo com a característica de "apenas leitura". Para trocar a identidade de uma ENTIDADE, deve-se,  portanto, cloná-la e alterar a identidade na construção;
  - `equals(entity)`: testa a equivalência entre a identidade de duas ENTIDADE, se verdadeiro indica que os objetos testados são ENTIDADES e possuem exatamente a mesma identidade (`EntityID`);
  - `generateId()`: método protegido chamado no construtor de uma ENTIDADE quando a sua identidade não for enviada como parâmetro. Esse método deve ser substituído de acordo com o objeto de identidade que é usado e criado pela ENTIDADE.
  - `emitter`: possuí uma propriedade protegida que é um objeto do tipo `EventEmmiter`. Este objeto deve ser usado para emitir eventos da ENTIDADE (`Entity.emit(event, ...args)`) e ouvir (`Entity.on(event, (...args) => void)`). Também é possível desinscrever um handler (`Entity.off`) ou desinscrever todos (`Entity.dispose()`).
  
  - Possuí uma flag protegida (`_modified`) para identificar se a ENTIDADE foi modificada ou não;
  - Possuí método `isModified()` para retornar o valor da flag;
  - Possuí métodos protegidos para gerenciamento de modificações. O `markAsModified()` para marcar a ENTIDADE como modificada (irá emitir o evento `modified` e marcar a flag `_modified`); e, `markAsPersisted()` para desmarcar (irá emitir o evento `persisted` e desfazer a flag).
  
- `OptionalEntity<Entity extends IEntity<ID>, ID extends EntityID<any> = EntityID<any>>` é uma implementação que sempre carrega a identidade da ENTIDADE, mas pode ou não carregar o objeto `Entity` hidratado. Esse objeto é útil para um “lazy loading” das ENTIDADES. Você carrega a identidade da ENTIDADE, mas não o carregamento da ENTIDADE é opcional. Este carregamento pode ser feito no construtor ou utilizando o método `safeLoad(entity)`.
  - Possuí o método `load(entity)` que reidrata a ENTIDADE, mas retorna um erro caso a identidade não seja a mesma. Para retornar um `Result`, deve-se utilizar `safeLoad(entity)`;
  - Possuí métodos para verificar se a ENTIDADE está presente ou não (`isPresent()` e `isAbsent()`);
  - Possuí um `getter` para forçar o retorno da ENTIDADE (`knowableEntity`). Isso garante que, caso a ENTIDADE não esteja hidratada, uma exceção seja lançada para a aplicação;

##### Coleções

> Todos os objetos de ENTIDADE podem ser armazenados em uma coleção aprimorada como `CollectionOfEntities`. Os métodos disponíveis para as coleções estão descritos abaixo.

- Propriedades:
  - `arrayOf`: retorna todos os itens como `array` de `OptionalEntity`;
  - `entries`: retorna `Iterator` de pares `[key, OptionalEntity]` da coleção;
  - `ids`: retorna `array` com todos as identidades das ENTIDADES;
  - `keys`: retorna `array` com todas as chaves da coleção. As chaves são geradas a partir da conversão das identidades para `string` com o método `EntityID.toString()`;
  - `knowableEntities/entities`: retorna `array` apenas das ENTIDADES que estão presentes (hidratadas);
  - `length`: retorna o número total de itens na coleção;
  - `values`: retorna `Iterator` com todos os valores `OptionalEntity`;
- Métodos
  - `add(item)`: adiciona uma ENTIDADE à coleção (não recarrega se já existir, usar o método `sync` para esse comportamento);
  - `addMany(items)`: adiciona múltiplas ENTIDADES de uma vez;
  - `appendRaw(item)`: anexa um `OptionalEntity` diretamente, sem verificação, substituindo sempre;
  - `appendManyRaw(items)`: anexa múltiplos `OptionalEntity` diretamente;
  - `sync(item)`: sincroniza uma ENTIDADE, sempre substituindo se já existir;
  - `syncMany(items)`: sincroniza múltiplas ENTIDADES de uma vez;
  - `find(id)`: busca uma ENTIDADE pelo ID, retorna a ENTIDADE ou `undefined`;
  - `forceFind(id)`: busca uma ENTIDADE pelo ID, retorna a ENTIDADE. O comportamento é forçado, ou seja, irá lançar uma exceção se não for encontrado. Deve ser usado quando você souber que a ENTIDADE existe;
  - `get(id)`: obtém um `OptionalEntity` pelo ID;
  - `getKey(key)`: obtém um `OptionalEntity` pela chave;
  - `has(id)`: verifica se a coleção possui uma ENTIDADE com o ID especificado;
  - `hasAll(ids)`: verifica se a coleção possui todas as ENTIDADES pelos IDs;
  - `hasAllItems(items)`: verifica se a coleção possui todas as ENTIDADES especificadas;
  - `hasAllKeys(keys)`: verifica se a coleção possui todas as chaves especificadas;
  - `hasAny(ids)`: verifica se a coleção possui alguma das ENTIDADES pelos IDs;
  - `hasAnyItems(items)`: verifica se a coleção possui alguma das ENTIDADES especificadas;
  - `hasAnyKeys(keys)`: verifica se a coleção possui alguma das chaves especificadas;
  - `hasItem(item)`: verifica se a coleção possui a ENTIDADE especificada;
  - `hasKey(key)`: verifica se a coleção possui a chave especificada;
  - `itemAvailableFor(id)`: verifica se a ENTIDADE está disponível (presente) para o ID;
  - `remove(id)`: remove uma ENTIDADE pelo ID;
  - `removeItem(item)`: remove uma ENTIDADE específica;
  - `removeKey(key)`: remove uma ENTIDADE pela chave;
  - `reload(item)`: recarrega uma ENTIDADE que já existe na coleção, se não existir lança uma exceção;
  - `reloadMany(items)`: recarrega múltiplas ENTIDADES existentes na coleção, se uma das ENTIDADES não existir lança uma exceção;
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

#### AGREGADOS

### OBJETO DE VALOR (`ValueObject`)

| Rótulo         | Descrição                                                    |
| -------------- | ------------------------------------------------------------ |
| Camada         | Domínio                                                      |
| Pergunta-chave | Corresponde a um atributo que pode ser descartável, substituível e imutável? |
| Comportamento  | Imutabilidade de valores. O objeto sempre será o mesmo e será equivalente a outro com os mesmos atributos. |

OBJETOS DE VALOR são objetos de domínio que **não** possuem uma identidade. Eles são descartáveis e equivalentes quando os atributos são mesmos para dois objetos. Em outras palavras, **Cor**, por exemplo, é um objeto de valor que pode ter um hexadecimal e um nome. Independente de quantas instâncias de `{ name: "White", hex: "#fff" }` sejam criadas, todas elas serão as mesmas uma vez que todos os seus atributos são iguais.

> É importante determinar o contexto para projetar OBJETOS DE VALOR. À exemplo, “endereço” pode ser um OBJETO DE VALOR para um pedido em um e-commerce, uma vez que é um atributo complementar ao pedido. Mas para um sistema de logística, cada endereço é devidamente catalogado e deve possuir uma identidade única, neste caso a classificação correta seria: endereço como ENTIDADE.

#### Características

- OBJETOS DE VALOR sempre são imutáveis. Esse comportamento permite um compartilhamento seguro do OBJETO DE VALOR com diferentes entidades, sem afetar nenhuma delas. Também torna o controle de alterações simples. Se um atributo do OBJETO DE VALOR precisar mudar, então, devemos criar um novo OBJETO DE VALOR com os novos atributos. A imutabilidade evita efeitos colaterais e problemas de identidade;
  - Em algumas instâncias, por exemplo, você pode ter o OBJETO DE VALOR `StatusValueObject` (que descreve o estado de um pedido, por exemplo). Mesmo que você carregue uma quantidade infinita de ENTIDADES, todas elas podem compartilhar os mesmos 5 estados (apenas 5 instâncias na memória). Mesmo que uma ENTIDADE altere seu próprio estado, a imutabilidade vai garantir que as cópias compartilhadas não sejam modificadas.

- Comparação por valor intrínseco: a igualdade entre dois OBJETOS DE VALOR é determinada sempre comparando seus atributos. Se todos eles forem iguais, então os objetos são os mesmos (mesmo que sejam instâncias diferentes na memória);
- São descartáveis ou substituíveis: pense nos OBJETOS DE VALOR como tipos primitivos aprimorados do domínio. Assim como você pode alterar um número (que permanece como um número), você pode alterar um OBJETO DE VALOR por outro do mesmo tipo;
- Auto-validação: OBJETOS DE VALOR não guardam regras de negócios, entretanto eles costumam garantir que os seus atributos passem por uma validação intrínseca. Por exemplo, um OBJETO DE VALOR “E-mail”, deve validar se o e-mail é correto durante a construção do mesmo;
- A mutabilidade dos OBJETOS DE VALOR geralmente é controlada pela ENTIDADE a qual ele pertence. Quando não for o caso, a ENTIDADE já deve exigir um OBJETO DE VALOR do tipo certo. Exemplo: é possível ter o método `changeColor(name, hex)` onde a ENTIDADE é responsável por criar o novo OBJETO DE VALOR ou o método `changeColor(vo)` que já recebe o OBJETO DE VALOR “Cor” já criado.

#### Boas Práticas

- Ao invés de utilizar o construtor padrão, utilizar um método estático dentro do próprio OBJETO DE VALOR (`ValueObject.create`) para atuar como `Factory` ou criar uma classe específica de montagem do OBJETO DE VALOR. Neste método (ou classe), você deve retornar um objeto `Result` ao invés de lançar uma exceção caso algum parâmetro não seja compatível;
- OBJETOS DE VALOR podem possuir métodos de mutabilidade, porém esses métodos devem retornar um novo OBJETO DE VALOR para ser reatribuído. Por exemplo, `Money.add(money)` irá retornar um novo OBJETO DE VALOR com o valor monetário atualizado. Preferencialmente, ao aplicar a mutação, retornar um objeto `Result` com o erro relacionado. Isso garante que exceções sejam lançadas apenas por comportamentos inesperados e não regras do domínio;
- OBJETOS DE VALOR devem ser preferencialmente transmitidos como parâmetros em mensagens entre objetos (exemplo, Serviço x ENTIDADE). Também deve ser usados como atributos de ENTIDADES (preferencialmente para valores com comportamentos complexos e reutilizáveis);
- OBJETOS DE VALOR devem ser conceitualmente completos;
- Garanta idempotência onde for apropriado, isto é: se uma ação não altera o estado, não falhe e não execute.

#### Mutabilidade

Se um OBJETO DE VALOR mudar frequentemente, a criação ou exclusão for cara, a substituição perturbar agrupamentos ou não existem benefícios de compartilhamento evidentes é possível optar pela utilização de ATRIBUTOS DE VALOR (`Attribute`). Os ATRIBUTOS DE VALOR são objetos similares aos OBJETOS DE VALOR com a diferença que podem ser mutáveis.

##### Boas Práticas

- Mesmo que ATRIBUTOS DE VALOR sejam mutáveis, é uma boa prática não expor para “fora” da entidade a mutabilidade direta do Atributo. À exemplo, suponha que você tem um ATRIBUTO DE VALOR de “Metadados” (`MetadataAttribute`). Mesmo que o ATRIBUTOS tenha os métodos para adicionar, remover e obter metadados, a ENTIDADE deve fazer um “proxy” para este Atributo ao invés de disponibilizá-lo diretamente. Desse modo, a ENTIDADE passa a ter os métodos: `addMetadata`, `removeMetadata` e `getMetadata`.

#### Implementação

> Os objetos abaixo estão disponíveis na biblioteca `@piggly/ddd-tookit`. Um OBJETO DE VALOR deve sempre estender `ValueObject`. Enquanto um Atributo deve sempre estender `Attribute`.

`ValueObject<Props extends Record<string, any> = Record<string, any>> implements IValueObject<Props>` é uma classe que recebe propriedade devidamente tipadas e as congela com `Object.freeze` impedindo a mutabilidade. Outros métodos também estão disponíveis:

- `equals(vo)`: retorna se os OBJETOS DE VALOR são idênticos;
- `hash()`: retorna o hash `sha256` do OBJETO DE VALOR;

##### Coleções

> Todos os OBJETOS DE VALOR podem ser armazenados em uma coleção aprimorada como `CollectionOfValueObjects`. Os métodos disponíveis para as coleções são similares à coleção de ENTIDADES.

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

### REPOSITÓRIOS (`Repository`)

| Rótulo         | Descrição                                                    |
| -------------- | ------------------------------------------------------------ |
| Camada         | Domínio / Infraestrutura (contratos no Domínio; implementações na Infraestrutura) |
| Pergunta-chave | **Preciso hidratar um AGREGADO completo ou só ler dados para exibição?** |
| Comportamento  | `WritableRepository`: hidrata/salva AGREGADOS. `ReadableRepository`: retorna **`DTOs`**. |

REPOSITÓRIOS abstraem a persistência e leitura de dados. Na implementação desta arquitetura, em conformidade com o padrão **CQRS**, os REPOSITÓRIOS, separam-se em:

- **`<Name>WritableRepository`**: operações de **escrita** e carregamento de AGREGADOS para aplicar regras/invariantes. Por exemplo, `OrderWritableRepository` é responsável por escrever todo o AGREGADO de pedidos em um banco de dados;
  - REPOSITÓRIOS de escrita podem ter implementações de leitura, por exemplo: `OrderWritableRepository.getById`. Neste caso, ao chama o método `getById` todo o AGREGADO do pedido será populado. Isso será essencial para fazer novas operações de escrita sobre ele. Desse modo, implementações de leitura só devem ser implementadas aqui quando forem essenciais para operações de escrita posteriores.
- **`<Name>ReadableRepository`**: operações de **leitura** que retornam `DTOs` (sem hidratar AGREGADOS). Os métodos de leitura podem ter **paginação/filtros** eficientes. Por exemplo, `OrderReadableRepository` é responsável por ler os dados do AGREGADO, sem hidratar uma entidade;
  - Um exemplo eficiente de uso é quando a aplicação deseja listar os pedidos com apenas algumas informações, como: ID, nome do cliente, data do pedido e status. Não é necessário carregar a entidade completa e o esquema de dados retornado deve ser compatível com o que é esperado para visualização. Geralmente esse dado é encapsulado em um `DTO`.

#### Características

- A implementação do REPOSITÓRIO fica na  Camada do Domínio da aplicação. Por outro lado, as implementações de banco de dados (sejam quais forem) vivem na Camada de Infraestrutura da Aplicação. Sendo assim, o driver do banco de dados será injetado como dependência do REPOSITÓRIO;

- O REPOSITÓRIO de Escrita sempre irá carregar o AGREGADO completo (de modo a respeitar as invariantes) e também salvará esse AGREGADO como uma unidade;

- O REPOSITÓRIO de Leitura nunca deve retornar nem entidades e muito menos AGREGADOS. Sempre irá retornar `DTOs` que podem conter dados parciais (ou completos) de um `snapshot` de uma entidade;

- Ambos tipos de REPOSITÓRIOS tem a finalidade de isolar a tecnologia (driver de implementação do banco de dados) da  Camada do Domínio através da injeção de dependências;
- Não valida regras de negócio. O REPOSITÓRIO deve receber parâmetros já padronizados e executar as operações. Desse modo, o REPOSITÓRIO faz defesa mínima (`null/undefined`, ou parâmetros que afetam a estabilidade query);
- A implementação de infraestrutura que é responsável por lançar erros quando algo der errado. O REPOSITÓRIO deve propagar esses erros. Mas ao resolver a query, por outro lado, o REPOSITÓRIO deve retornar `undefined` quando não conseguir obter uma resultado ou `false` para casos em que não seja necessário retornar dados explícitos.

#### Boas Práticas

- Um REPOSITÓRIO não deve “vazar” detalhes da infraestrutura utilizada (queries, drivers, conexões, etc) para o domínio;
- Nos métodos de leitura, sempre exponha métodos orientados a casos de uso (por exemplo, `getById(EntityID)`). Evite utilizar CRUD genérico ao desenhar os métodos (por exemplo, `get`, `create`, `update`, etc). Seja explícito;
- Dentro de um REPOSITÓRIO de Escrita, sempre coloque as hidratações necessárias para os métodos de estrita para garantir que as invariantes das ENTIDADES ou AGREGADOS manipulados sejam respeitadas;
- Nunca retorne ENTIDADES em um REPOSITÓRIO de Leitura;
  - É preferencial que os parâmetros dos REPOSITÓRIOS de Leitura sejam um OBJETO DE VALOR (`ValueObject`) específico, montado na Camada da Aplicação, antes de chamar o REPOSITÓRIO;
  - Os OBJETOS DE VALOR utilizados devem ser infra-agnósticos, eles não devem saber nada que seja relacionado ao banco de dados. Por exemplo, o objeto `PaginationValueObject` não sabe nada sobre `offset`/`limit`, ele sabe sobre `page`/`size`. Além disso, esse objeto não faz a conversão dos valores, quem faz isso é o REPOSITÓRIO.
- Sempre retorne ENTIDADES em um REPOSITÓRIO de Escrita.
  - É preferencial que os parâmetros dos REPOSITÓRIOS de Escrita sejam uma ENTIDADE (`Entity`) ou uma Identidade da ENTIDADE (`EntityID`).

#### Implementação

> A biblioteca `@piggly/ddd-tookit` não força uma implementação para REPOSITÓRIOS. Esteja livre para criar. Você pode seguir as regras abaixo para se organizar melhor.

- No construtor de um REPOSITÓRIO, injete todas as dependências de infraestrutura;
- Crie métodos independentes e isolados. Deixe a Camada da Aplicação chamar as operações necessárias, como: carregar o AGREGADO, fazer as alterações e então salvar o AGREGADO.

### SERVIÇOS DE DOMÍNIO (`DomainService`)

| Rótulo         | Descrição                                                    |
| -------------- | ------------------------------------------------------------ |
| Camada         | Domínio                                                      |
| Pergunta-chave | A regra depende de múltiplos AGREGADOS ou não “cabe” naturalmente em um só? |
| Comportamento  | Políticas de negócio coesas e reutilizáveis, preferencialmente **puras** (sem I/O). |

SERVIÇOS DE DOMÍNIO encapsulam regras e políticas de negócio que:

- Dependem de mais de um AGREGADO (ou não pertencem claramente a um único);
- Precisam ser reutilizadas por várias ENTIDADES ou Handlers;
- Podem ser puras (sem I/O). Quando dependerem de I/O, deve-se injetar portas (interfaces) do domínio.

#### Características

- Não possuem estado, isso significa que são um conjunto de funções (relacionadas) puras. Um dado entra, o resultado saí;
- Focam em política sempre, não em orquestração. A orquestração é uma responsabilidade da Camada da Aplicação;
- São reutilizáveis, ou seja, podem ser chamados por ENTIDADES e Handlers sem duplicar lógica.

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
- Os parâmetros de um método implementável são livres para receber um OBJETO DE VALOR ou uma ENTIDADE (não pode sofrer mutação no serviço). O objetivo é apenas ler os dados e retornar os resultados de uma operação. Se você aplica uma política de cálculo de desconto, você pode ler a ENTIDADE de Pedido, mas deverá retornar o cálculo independente. A  Camada da Aplicação que decide o que fazer com o dado retornado;
- O nome do serviço deve respeitar a Linguagem Ubíqua, sempre se referindo a política implementada;
- Não deve depender de infraestrutura (serviços externos), apenas interfaces do domínio. Todos os adaptadores ficam na infra.

#### Regras de Negócio: no Serviço ou na ENTIDADE?

Para decidir se uma regra de negócio deve fazer parte de um SERVIÇO DE DOMÍNIO ou de uma ENTIDADE. Você deve percorrer o checklist abaixo e garantir o posicionamento correto das regras:

1. A regra aplicável depende de mais de um AGREGADO (ou entidades distintas)? Se aplicável, então SERVIÇO DE DOMÍNIO;
2. A regra depende de infraestrutura? Se aplicável, então SERVIÇO DE DOMÍNIO;
3. A regra é responsável por uma invariável interna do AGREGADO? Se aplicável, então na raiz do AGREGADO ou no OBJETO DE VALOR;
4. É um cálculo puro sobre os valores sem identidade? Se aplicável, no OBJETO DE VALOR ou uma função pura;
5. É uma orquestração de passos (carregar entidades, chamar domínios, persistir, etc)? Se aplicável, então Serviço de Aplicação;
6. É uma política “plugável” ou frequentemente mutável? Se aplicável, então SERVIÇO DE DOMÍNIO.

#### Implementação

> A biblioteca `@piggly/ddd-tookit` não força uma implementação para SERVIÇOS DE DOMÍNIO, embora você possa estender `DomainService` apenas para fins de classificação. Esteja livre para criar. Você pode seguir as regras abaixo para se organizar melhor.

- No construtor de um SERVIÇO DE DOMÍNIO, injete todas as dependências externas como interfaces do Domínio ou operações de I/O.

### Eventos de Domínio (`DomainEvent`)

| Rótulo         | Descrição                                                    |
| -------------- | ------------------------------------------------------------ |
| Camada         | Domínio (emissão); Aplicação (assinatura/processamento).     |
| Pergunta-chave | Um fato relevante do domínio ocorreu que outros componentes devem saber? |
| Comportamento  | Fato imutável publicado quando uma operação válida altera o estado do domínio. |

Eventos de Domínio narram **fatos** (por exemplo, `OrderPaidDomainEvent`). São emitidos pelo AGREGADO quando uma mudança válida ocorre. A camada de aplicação os **ouve** para disparar ações (por exemplo, notificar o cliente), mantendo baixo acoplamento.

> Não confundir com Eventos Locais de uma ENTIDADE. Os eventos Locais de uma entidade servem para propagar mudanças de estado entre partes da ENTIDADE. Por outro lados, Eventos de Domínio são um fato concreto que pode ser capturado pela aplicação e direcionado para ações específicas.

> É recomendado usar a biblioteca [@piggly/event-bus](https://www.npmjs.com/package/@piggly/event-bus). A biblioteca está preparada para receber e disparar eventos de qualquer lugar da aplicação.

#### Características

- Os Eventos de Domínio devem ser imutáveis e descritivos, além disso o nome deve estar sempre no passado;
- Os AGREGADOS são responsáveis por emitir os eventos, enquanto os Handler e Gerenciadores de Processos devem reagir;
- Mantem baixo acoplamento, uma vez que os produtores não conhecem e não se importam com os consumidores;
- A entrega é no mínimo “best effort”; para robustez é recomendado usar uma **Outbox**.

#### Boas práticas

- Nomeie o evento com o fato concreto (ubíquo);
- Os parâmetros do evento devem ser montados preferencialmente estendendo `EventPayload` da biblioteca `@piggly/event-bus`;
- Emitir na raiz do AGREGADO quando o fato acontece;
- Não colocar regras de negócio no handler do evento, use-o apenas para orquestrar (enviar para uma Queue, por exemplo);
- Utilizar Outbox se precisar garantir publicação após commit.

### Considerações Gerais

Ainda existem alguns conceitos de domain-driven design que devem ser implementados no fluxo da aplicação:

- **Fábricas (`Factory`)**: Fábricas são úteis para criar AGREGADOS complexos. Em vez de ter `new OrderAggregateRoot(...)` exposto, poderíamos ter um método estático `OrderAggregateRoot.create(args)` ou uma classe `CreateOrderFactory` que monta um pedido válido. A Fábrica encapsula o processo de montagem respeitando invariantes;

- **Contextos Delimitados (Bounded Contexts)**: DDD estratégico reconhece que um grande sistema pode ser dividido em múltiplos subdomínios, cada um com seu modelo. Nosso exemplo é focado em um contexto (vendas). Em um sistema real, o **Contexto de Vendas** (pedidos, produtos, pagamentos) poderia ser distinto do **Contexto de Catálogo** (gerência de produtos, categorias) ou de **Usuários**. Isso implica que as mesmas palavras podem ter significados diferentes em contextos diferentes. Bounded contexts são normalmente implementados como módulos ou microsserviços separados;

- **Linguagem Ubíqua**: Ao implementar, sempre use nomes em inglês e contextuais. Isso é deliberado para espelhar a linguagem do negócio. É necessário manter a consistência com os termos de domínio usados pelos stakeholders e aplicáveis ao contexto. O importante é que todos usem os mesmos termos para evitar confusão;

- **Teste de Unidade e Design**: DDD encoraja desenho focado em domínio que costuma resultar em código mais testável. ENTIDADES e SERVIÇOS DE DOMÍNIO não dependem de infraestrutura, então podemos instanciá-los em testes simples. Por exemplo, testar `OrderAggregateRoot.addItem` invariantes, ou `DiscountPoliciesDomainService.calculate` sem precisar de banco ou HTTP. Isso leva a maior confiança no core do sistema.

- **Camada de Infraestrutura**: Ainda que não seja foco, lembre-se que eventualmente terá implementação de REPOSITÓRIOS (ex: usando Sequelize, TypeORM, mongoose, etc.), implementação real de envio de E-mail, logs, etc. Essas partes devem ficar de fora do domínio. Uma prática comum é usar **interfaces** ou classes abstratas na  Camada do Domínio para representar, por exemplo, um `IEmailService` genérico, e na infra ter `EmailService` com envio via SMTP ou API. Depois, injetar essa implementação na  Camada da Aplicação. Assim, se um dia mudar a forma de envio de e-mail, nada no domínio muda.

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
- Na  Camada do Domínio, ENTIDADES, OBJETOS DE VALOR e SERVIÇOS DE DOMÍNIO, podem retornar um `Result.fail` para regras esperadas (invariantes e validação);
  - `BusinessRuleViolationError` pode ser usado com extensão de erros relacionados a invariantes;
  - `InvalidPayloadSchemaError` pode ser usado como extensão de erros relacionados a validação.
- Na Camada da Aplicação, Mediador, Handlers, Comandos, Queries e Serviços da Aplicação, podem retornar um `Result.fail` para regras esperadas (invariantes e validação);
  - `BusinessRuleViolationError` pode ser usado com extensão de erros relacionados a invariantes do domínio;
  - `InvalidPayloadSchemaError` pode ser usado como extensão de erros relacionados a validação;
  - `ApplicationError` pode ser usado como extensão de erros gerais relacionados a aplicação.
- Na Camada de Infraestrutura, evitar usar o `Result`, neste caso, é preferível lançar exceções e, opcionalmente, estender o objeto `RuntimeError`. Na última ponta da aplicação, pode ser convertido para um erro de apresentação para o usuário, usando o erro verdadeiro apenas para logging e monitoramento.

#### Boas Práticas

- **Onde usar `Result`**:

  - Na  Camada do Domínio para invariantes e regras esperadas.

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
