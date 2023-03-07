---
layout: post
title: "C# - Threads, Process e Tasks [pt-BR]"
excerpt: "Entendendo process, threads e tasks em C#"
---

# CSharp - Entendendo Threads, Tasks e Process

Para entender threads e tasks em C# primeiro é importante entender alguns conceitos de computação com múltiplas threads.
Dentro da linguagem esses conceitos se apresentam através de algumas classes, mas não são nada exclusivos de C#, e sim conceitos de processamento em paralelo implementados pelos sistemas operacionais.

Visão geral:
![Visão geral](/assets/img/ProcessThreadsTasks.png)


# Process

O process se refere ao programa que está em execução. Quando você inicia uma aplicação, seja ela ASP.NET, Console ou Desktop, um novo process é criado.
O papel do process é separar os recursos utilizados na execução de cada programa e estabelecer "a fronteira" entre cada um, de forma que cada processo só consiga acessar seus próprios recursos.


# Thread

Uma thread é a menor unidade de processamento, ela possui um conjunto de instruções, uma porção de memória e todas as informações necessárias para executar um trecho de código.
Em um nível mais profundo, uma thread consiste em uma porção da capacidade de processamento da CPU, seus registradores, um conjunto de instruções do programa e uma porção da memória onde estão armazenados os valores necessários para execução da thread.

Um processo pode iniciar diversas threads, porém, é importante saber que iniciar e finalizar threads é um processo custoso! Por isso, deve ser usado com parcimônia.


# Mas como isso existe e é utilizado em C#?

Em C# nós temos classes para iniciar e gerenciar novos processos e novas threads, especificamente:

```csharp
using System.Diagnostics.Process;
using System.Threading.Thread;
```

A classe `Process` permite iniciar a execução de um novo programa em um processo separado.
Já a classe `Thread` permite a criação de uma nova thread dentro do processo em execução atualmente.

## Threads em C#.

Podemos criar novas threads para realizar processamento em paralelo.

Exemplo:
Vamos supor que eu precise obter um valor do banco de dados, eu posso fazer isso usando uma outra thread.

```csharp
long? valueFromDb = null;
var getFromDbThread = new Thread(() =>
{
	valueFromDb = dbService.GetValueFromDb();
});

getFromDbThread.Start();
```

O código acima cria uma nova thread, atribui à ela uma execução através de um delegate e no trecho `getFromDbThread.Start();` a execução é iniciada.
Threads não retornam valores, por isso, antes de iniciar a thread foi necessário criar uma variável que irá armazenar o valor, nesse caso se trata de `long? valueFromDb = null;`.

No exemplo acima, seria necessário verificar se a variável `valueFromDb` já foi preenchida em algum momento para saber se a thread finalizou a execução, ou podemos usar o comando `.Join()` para aguardar o fim da execução da thread.

```csharp
long? valueFromDb = null;
var getFromDbThread = new Thread(() =>
{
	valueFromDb = dbService.GetValueFromDb();
});

getFromDbThread.Start();

//faço outras coisas

getFromDbThread.Join(); //Aguardo a thread finalizar a execução.
```

Caso seja necessário encadear processamentos, é necessário verificar se o processo anterior finalizou para iniciar uma nova thread.

```csharp
long? valueFromDb = null;
var getFromDbThread = new Thread(() =>
{
	valueFromDb = dbService.GetValueFromDb();
});

getFromDbThread.Start();
getFromDbThread.Join(); //Aguardo a thread finalizar a execução.

//Inicio outra thread
long? anotherValueFromDb = null;
getFromDbThread = new Thread(() =>
{
	anotherValueFromDb = dbService.GetAnotherValueFromDb(valueFromDb);
});

getFromDbThread.Start();
getFromDbThread.Join(); //Aguardo a thread finalizar a execução.
```


## Lidando com exceções em threads

As threads não propagam exceções automaticamente, então, temos que gerenciar isso manualmente também:

```csharp
long? valueFromDb = null;
Exception exp? = null;

var getFromDbThread = new Thread(() =>
{
	try {
		valueFromDb = dbService.GetValueFromDb();
	}
	catch(Exception threadException) {
		exp = threadException;
	}
});

getFromDbThread.Start();
getFromDbThread.Join(); //Aguardo a thread finalizar a execução.

if(exp is not null)
	throw exp;
```

É necessário criar uma variável que irá receber e exceção e verificar se houve algum problema para propagar o erro.


# E é aí que entram as Tasks!

Como deu pra perceber, utilizar threads para processamento em paralelo tem muitos pontos negativos:
- Código muito verboso
- Complicado de gerenciar resultados
- Complicado de lidar com exceções
- Iniciar e finalizar uma nova thread é um processo custoso para o Sistema Operacional

Por causa disso foram criadas as Tasks!
Tasks são uma abstração em cima do conceito de threads, por baixo dos panos elas utilizam threads, mas possuem uma interface muito mais amigável.

As tasks apresentam diversas vantagens quando comparadas com threads para exeucução em paralelo:
- Podem retornar valores
- Propagam exceções por padrão
- É muito fácil gerenciar execução em paralelo e encadear várias tasks
- Tasks não iniciam uma nova thread a cada execução pois se utilizam da ThreadPool (falaremos mais disso abaixo)

## Retornando valores com tasks

```csharp
var valueFromDbTask = Task.Run(() => dbService.GetValueFromDb());

long valueFromDb = valueFromDbTask.Result;
```

O código se torna muito mais limpo e há um ganho de performance considerável.

## Lidando com exceções

```csharp
try {
	var valueFromDbTask = Task.Run(() => dbService.GetValueFromDb());
	
	long valueFromDb = valueFromDbTask.Result;
}
catch(AggregateException exceptions) {
	throw exceptions.InnerExceptions[0];
}
```

Tasks propagam exceções através do tipo AggragateException, que por sua vez possui uma array de `InnerExceptions` que pode ser verificada.

## Encadeando tasks

```csharp
var valueFromDbTask = Task.Run(() => dbService.GetValueFromDb());
valueFromDbTask.ContinueWith(anotherTask => dbService.GetAnotherValueFromDb(valueFromDbTask.Result));
```

Podemos usar `.ContinueWith()` para continuar uma task com outra e assegurar execução sequencial.

## Executando diversas tasks paralelamente

```csharp
var valueFromDbTask = Task.Run(() => dbService.GetValueFromDb());
var yetAnotherValue = Task.Run(()) => dbService.GetYetAnotherValueFromDb());

await Task.WhenAll(valueFromDbTask, yetAnotherValue);
```

Podemos usar `Task.WhenAll()` para executar várias tarefas de forma paralela.
Especialmente útil em casos onde são tarefas facilmente paralelizaveis, como enviar múltiplas requisições para obter dados.

## Mas existe algum cenário onde faça sentido usar Threads ao invés de Tasks?

Para entender isso, é interessante entender como as Tasks funcionam dentro da plataforma .NET.
As tasks são executadas em uma Thread Pool gerenciada (managed thread pool) que consiste em um conjunto de threads que são gerenciadas pelo próprio runtime do .NET.
Essas threads são ótimas para execução de diversas pequenas tarefas ao longo do processamento do programa, no entanto, como existe um número limitado de threads no thread pool, ocupar uma thread por um longo período de tempo pode ser prejudicial à performance do programa.

Por causa disso **iniciar manualmente Threads é uma opção melhor em casos onde a tarefa será executada por longos períodos de tempo**, como por exemplo uma tarefa em background que consome itens de uma fila. 

## Conclusão

No geral, tasks são a maneira recomendada de de se lidar com execução de tarefas em segundo plano. Elas fornecem um ferramental muito mais poderoso e produzem um código mais enxuto.
Criar threads manualmente é mais interessante para tarefas de longa duração para não comprometer as threads do thread-pool que podem ser usadas para tarefas menores.

---------
### Disclaimers

#### Sobre Async/Await
Async/Await são conceitos levemente diferentes dos tratados nesse artigo. É possível utilizar await para aguardar a execução de uma ou mais tarefas, e o principal efeito de utilizar async/await é que, enquanto o comando aguardado está sendo executado em outra thread, a thread atual fica livre para processar outras requisições.
Async/Await também cria um overhead de processamento/memória, então, do ponto de vista de consumo de recursos, executar tarefas de forma síncrona economiza mais recursos. **Porém**, utilizar async/await é benéfico pois utiliza melhor os recursos da máquina como um todo.

#### Sobre a diferença entre linguagem e implementação
É importante entender que a linguagem C# é algo separado da implementação. Os conceitos descritos acima só se aplicam à implementação de C# do CLR da Microsoft através da plataforma .NET 7.


