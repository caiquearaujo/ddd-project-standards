# Arquitetura de Projeto

Este documento contém uma série de padrões de projetos que devem ser seguidos e respeitados. A arquitetura, aqui descrita, pode ser implementada em qualquer linguagem. A fim de demonstração, a linguagem adotada será `Javascript` com o uso de tipagem do `Typescript`.

> **Recomendação**: o toolkit `@piggly/ddd-toolkit` pode ser utilizado para auxiliar na implementação, uma vez que o objetivo dessa biblioteca é manter os componentes padrões organizados.

## Camada de Domínio

### Entidade

| Rótulo         | Descrição                                                    |
| -------------- | ------------------------------------------------------------ |
| Camada         | Domínio                                                      |
| Pergunta-chave | A aplicação (ou usuário) precisa saber se este objeto é o mesmo de antes? |
| Comportamento  | Mutabilidade de atributos, permanência de identidade.        |

Objetos de domínio que possuem uma identidade contínua e única, independente da equidade dos atributos. Em outras palavras, **Cliente**, por exemplo, é uma entidade que possuí uma identidade com um código único (id, geralmente), pois sempre representará aquele mesmo indivíduo. Mesmo que hajam clientes com mesmo nome, ainda serão clientes diferentes por conta da identidade única de cada um. Mesmo que o Cliente altere seu e-mail, ainda fazemos referência ao mesmo cliente de antes.

#### Características

1. Possuem uma identidade única e exclusiva que não se altera ao longo do tempo: identificador auto incrementado, identificador aleatório (UUID), CPF de um indivíduo, CNPJ de uma empresa, SKU de um produto, etc;
2. Possuem mutabilidade de atributos: podem mudar quaisquer valores internos (a depender das regras de cada entidade), mas ainda permanecem sendo o mesmo objeto do ponto de vista do negócio. Exemplo: um pedido pode mudar de status e continua sendo o mesmo pedido;
3. Duas entidades com identidades diferentes não são e não devem ser consideradas iguais, mesmo que todos os atributos sejam semelhantes ou equivalentes. Exemplo: usuários com mesmo nome, ainda são usuários diferentes se possuírem identidades diferentes.

#### Implementação

> Os objetos abaixo estão disponíveis na biblioteca `@piggly/ddd-tookit`.

- `EntityId<Value = string>`: objeto usado por uma entidade para guardar o valor da identidade. O valor dessa identidade pode ser uma `string`, um `number` ou um objeto complexo como `ObjectId`. Quando um novo `EntityId` é criado (sem uma carga inicial), ele é iniciado com um valor aleatório do mesmo tipo esperado para a entidade. Esse valor, posteriormente pode ser reidratado pelo banco de dados;
  - `equals(entityId)`: testa a equivalência entre o identificador da entidade, se verdadeiro, por tanto, significa que o identificador é o mesmo entre dois `EntityId` criados;
  - `isRandom()`: retorna se a identidade ainda está com valor aleatório (caso verdadeiro) ou se já é uma identidade fixa e persistente (caso falso);
  - `toNumber()`: força a conversão da identidade para o valor primitivo do tipo `number`;
  - `toString()`: força a conversão da identidade para o valor primitivo do tipo `string`;
  - `generateRandom()`: utilizado para implementar a geração de um valor aleatório para identidade. Ao criar um modelo de identidade, é recomendado substituir esse método. Ele deve ser chamado no construtor caso nenhum valor de identidade seja enviado como parâmetro.
  - Objetos mais literais podem ser utilizados, como: `StringEntityId`, `NumberEntityId` e `UUIDEntityId`.

- `Entity<Props, Id extends EntityID<any>>`: encapsula a entidade (com a sua identidade) em um objeto. A entidade pode ter qualquer conjunto de atributos (`Props`) e qualquer tipo de identidade compatível com `EntityId`;
  - O `_id` e `_props` são atributos do objeto que não devem estar acessíveis, por tanto devem são virtualmente protegidos (`protected`) para evitar acesso direto. Ainda, o `_id` deve ser um atributo com a característica de "apenas leitura". Para trocar a identidade de uma entidade, deve-se, por tanto, cloná-la e alterar a identidade na construção;
  - `equals(entity)`: testa a equivalência entre duas entidade, se verdadeiro indica que os objetos testados são entidades e possuem exatamente a mesma identidade (`EntityId`);
  - `generateId()`: método protegido chamado no construtor de uma entidade quando a sua identidade não for enviada como parâmetro. Esse método deve ser substituído de acordo com o objeto de identidade que é usado e criado pela entidade.
  - `emitter`: possuí uma propriedade pública que é um objeto do tipo `EventEmmiter`. Este objeto deve ser usado para emitir eventos da entidade (`emit(event, ...args)`) e ouvir (`on(event, (...args) => void)`). Também é possível desinscrever um handler (`off`) ou deinscrever todos (`unsubscribeAll()`).
- `EnhancedEntity<Props extends { updated_at: moment.Moment }, Id extends EntityID<any>>`: é um objeto alternativo e aprimorado para criar entidades que estende `Entity`. Ele exige que a entidade possua uma propriedade `updated_at` do tipo `moment.Moment`. Desse modo ele pode gerenciar mudanças na entidade;
  - Possuí uma flag protegida (`_modified`) para identificar se a entidade foi modificada ou não;
  - Possuí método `isModified()` para retornar o valor da flag;
  - Possuí o método `markAsModified()` para marcar a entidade como modificada (irá emitir o evento `modified` e marcar a flag `_modified`) e outro método `markAsPersisted()` para desmarcar (irá emitir o evento `persisted` e desfazer a flag).