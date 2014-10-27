---
layout: post
title: "Manutenção de schema de banco de dados com migrations em .NET"
date: 2014-10-26 19:55:00 -0200
comments: true
categories: [integracao continua, ci, .net, jenkins, dicas]
keywords: integracao continua, jenkins, .net, ci, continuous integration, dicas
description: Veja algumas dicas sobre Integração Contínua para .NET utilizando o Jenkins.
---

A manutenção de schema de banco de dados é algo importante e delicado em qualquer projeto de desenvolvimento de software. Quando o assunto é banco de dados, todos os desenvolvedores do time precisam estar na mesma página para que possamos realizar mudanças rápidas e com o mínimo de propensão a falhas possível. Com o intuito de atingirmos esses objetivos é importante que o schema do banco de dados esteja protegido pelo nosso controle de versão, e que de preferência, faça parte do processo de Integração Contínua do projeto.

Normalmente o processo de atualização e manutenção de schema de banco de dados é realizado de maneira manual através da criação de scripts SQL e/ou utilização de softwares que geram estes scripts a partir da comparação de dois bancos de dados. Apesar deste processo funcionar em muitos projetos existentes, ele exige muita intervenção manual, o que pode acabar atrasando o lançamento de uma nova versão do sistema e prejudicando os processos de Integração Contínua do projeto.

Uma maneira interessante de abordar a manutenção de schema de banco de dados é através da utilização de migrations. Neste post irei explanar um pouco sobre migrations e demonstrar uma abordagem de como utilizá-las em projetos .NET através de um ótimo framework, o <a href="https://github.com/schambers/fluentmigrator" target="_blank">FluentMigrator</a> 

<!--more-->

#O Básico Sobre Migrations

As migrations surgiram como uma alternativa as alterações de banco de dados através da escrita de SQL, bem como uma maneira de evoluir o schema de banco de dados de uma forma gradativa, maneira essa muito válida quando se adota metodologias ágeis. As migrations normalmente são escritas em uma linguagem de programação mais simples para descrever as mudanças no banco de dados, o que normalmente facilita a escrita e a leitura das mesmas.

Esse conceito de migrations foi popularizado pela comunidade <a href="http://rubyonrails.org/" target="_blank">Ruby on Rails</a> que o aplica através das <a href="http://guides.rubyonrails.org/migrations.html" target="_blank">Active Record Migrations</a>, de onde o FluentMigrator buscou inspiração.

##Implementando o FluentMigrator

A abordagem que irei demonstrar neste post é a que tenho utilizado e recomendado atualmente, mas ela é apenas uma das possíveis maneiras de utilização do FluentMigrator, fique a vontade para implementar da maneira que achar mais interessante para os seus projetos.

Para auxiliar este post eu desenvolvi um pequeno projeto de exemplo que pode ser encontrado em <a href="https://github.com/marcelobalexandre/fluentmigrator-poc" target="_blank">github.com/marcelobalexandre/fluentmigrator-poc</a>. 

Como é possível verificar na imagem abaixo eu organizei a estrutura do projeto de exemplo da seguinte maneira:

- **Scripts**: nesta pasta são mantidos os scripts SQL, nesse caso temos o de criação do banco de dados e um script de criação de tabelas antes da implementação do FluentMigrator.
- **MyProject.Data.Migrations**: neste projeto temos as migrations organizadas em pastas com o número da versão do projeto a que se referem e o MyProjectMigrator que é responsável por executar as migrações.
- **MyProjectMigratorRunner**: este projeto é um Console Application que utiliza o MyProjectMigrator para executar as migrações.

{% img /images/manutencao-de-schema-de-banco-de-dados-com-migrations-em-net/solution-explorer.png %}

Minha recomendação é criar um projeto separado que conterá as migrations e o nosso Migrator, que no projeto de exemplo chamei de MyProject.Data.Migrations. Basta criar uma Class Library e instalar o Fluent Migrator e o Fluent Migrator Runner utilizando o NuGet.

##Criando um Migrator Runner

Para executarmos as migrations podemos utilizar o Command Line Runner (executável do próprio FluentMigrator), o MSBuild, o Rake, ou outros. Eu particularmente prefiro criar um Runner próprio para o projeto, que neste caso chamei de MyProjectMigrator.

Como é possível observar no código abaixo, o MyProjectMigrator é bastante simples e fácil de usar, basta invocar o método `Migrate` passando como parâmetros a ação que deseja realizar (Up ou Down), a string de conexão com o banco de dados e o MigratorEnvironment. O MigratorEnvironment é um Enumerable que criei para que você possa executar ações diferentes nas migrations dependendo do ambiente, muito útil para por exemplo, criar dados fakes (seeds) no ambiente de desenvolvimento e realizar ações exclusivas para os testes unitários. É importante observar que neste exemplo o MyProjectMigrator espera uma string de conexão para o SQL Server 2012, se o seu caso é diferente basta alterá-lo.

``` c# MyProjectMigrator.cs
using FluentMigrator;
using FluentMigrator.Runner;
using FluentMigrator.Runner.Announcers;
using FluentMigrator.Runner.Initialization;
using FluentMigrator.Runner.Processors.SqlServer;
using System;
using System.Reflection;

namespace MyProject.Data.Migrations
{
    public enum MigratorEnvironment
    {
        Development,
        Test,
        Production
    }

    public static class MyProjectMigrator
    {
        private class MigrationOptions : IMigrationProcessorOptions
        {
            public bool PreviewOnly { get; set; }

            public int Timeout { get; set; }

            public string ProviderSwitches
            {
                get { throw new NotImplementedException(); }
            }
        }

        public static void Migrate(Action<IMigrationRunner> migrationRunnerAction,
                                   string connectionString,
                                   MigratorEnvironment migratorEnvironment)
        {
            Assembly assembly = Assembly.GetExecutingAssembly();

            var textWriterAnnouncer = new TextWriterAnnouncer(write => Console.WriteLine(write));
            var runnerContext = new RunnerContext(textWriterAnnouncer);
            runnerContext.ApplicationContext = migratorEnvironment;

            var migrationOptions = new MigrationOptions { PreviewOnly = false, Timeout = 0 };
            var processorFactory = new SqlServer2012ProcessorFactory();
            var processor = processorFactory.Create(connectionString,
                                                    textWriterAnnouncer,
                                                    migrationOptions);

            var migrationRunner = new MigrationRunner(assembly, runnerContext, processor);
            migrationRunnerAction(migrationRunner);
        }
    }
}
```

##Criando as Migrations

Recomendo a leitura da <a href="https://github.com/schambers/fluentmigrator/wiki/Migration" target="_blank">documentação oficial</a> sobre a criação de migrations que é bem direta e de fácil entendimento, mas basicamente para criamos uma migration é preciso criar uma nova classe que derive da classe abstrata `Migration` e implemente dois métodos, o `Up` e o `Down`. Além disso você devera adicionar o atributo Migration que aceita valores Int64, esse atributo será utilizado para identificar a migration, é baseado nesse atributo que o FluentMigrator vai saber se já rodou ou não a migration.

A classe abstrata `Migration` nos fornece diversos métodos para manipulação de banco de dados, o que nos permite criar tabelas, criar colunas, renomear colunas, inserir dados, executar um script SQL, dentre outras tarefas. Para conhecer mais e entender melhor esses métodos recomendo a leitura da <a href="https://github.com/schambers/fluentmigrator/wiki/Fluent-Interface" target="_blank">documentação oficial</a> sobre Fluent Interface.

Algo para se ter em mente na hora de criar uma migration é que não devemos agrupar muitas alterações no banco de dados em apenas uma migration, se você precisa criar duas tabelas novas por exemplo, faça isso em duas migrations  separadas e não em apenas uma. Outra boa prática é em relação aos nomes das migration, adote um padrão claro para que todo o time entenda de maneira fácil o que determinada migration irá fazer apenas olhando o nome da mesma, se sua migration irá criar uma tabela Clients por exemplo, uma boa opção seria CreateClientsTable. Outra dica é utilizar o padrão YYYYMMDDHHMM no atributo identificador da migration, dessa maneira conseguimos saber rapidamente quando foi desenvolvida e diminuímos o risco de dois desenvolvedores criarem uma migration com o mesmo identificador.

Criei alguns exemplos de como uso e organizo migrations no projeto de exemplo, mas lembre-se que mais importante que usar esse ou aquele padrão, é utilizar algum e fazer com que o time todo o siga.

``` c# 01 - RunBaselineScript.cs
using FluentMigrator;
using System;
using System.IO;

namespace MyProject.Data.Migrations._1._0._0._0
{
    [Migration(01)]
    public class RunBaselineScript : Migration
    {
        public override void Up()
        {
            Execute.Script(Path.Combine(Environment.CurrentDirectory, @"Scripts\02 - BaselineScript.sql"));
        }

        public override void Down()
        {
        }
    }
}
```

``` c# 201410192200 - CreateTestTable.cs
using FluentMigrator;

namespace MyProject.Data.Migrations._1._0._1._0
{
    [Migration(201410192200)]
    public class CreateTestTable : Migration
    {
        public override void Up()
        {
            Create.Table("Test")
                  .WithColumn("Id").AsGuid().NotNullable().PrimaryKey();
        }

        public override void Down()
        {
            Delete.Table("Test");
        }
    }
}
```

``` c# 201410192210 - AddColumnTestToTableTest.cs
using FluentMigrator;

namespace MyProject.Data.Migrations._1._0._1._0
{
    [Migration(201410192210)]
    public class AddColumnTestToTableTest : Migration
    {
        public override void Up()
        {
            Alter.Table("Test")
                 .AddColumn("Test").AsString(255).NotNullable();
        }

        public override void Down()
        {
            Delete.Column("Test").FromTable("Test");
        }
    }
}
```

``` c# 201410192215 - CreateTestTableSeedData.cs
using FluentMigrator;
using System;

namespace MyProject.Data.Migrations._1._0._1._0
{
    [Migration(201410192215)]
    public class CreateTestTableSeedData : Migration
    {
        public override void Up()
        {
            if ((MigratorEnvironment)this.ApplicationContext == MigratorEnvironment.Development)
            {
                Insert.IntoTable("Test").Row(new { Id = Guid.NewGuid(), Test = "Test 01" });
                Insert.IntoTable("Test").Row(new { Id = Guid.NewGuid(), Test = "Test 02" });
                Insert.IntoTable("Test").Row(new { Id = Guid.NewGuid(), Test = "Test 03" });
                Insert.IntoTable("Test").Row(new { Id = Guid.NewGuid(), Test = "Test 04" });
                Insert.IntoTable("Test").Row(new { Id = Guid.NewGuid(), Test = "Test 05" });
            }
        }

        public override void Down()
        {
        }
    }
}
```

##Executando as Migrations

Como visto anteriormente eu criei um Migrator Runner próprio chamado MyProjectMigrator, ele pode ser utilizado para rodar as migrations nos seus projetos de teste, em uma Console Application (como é o caso do projeto de exemplo) ou em um aplicativo com uma interface gráfica amigável. Como é possível observar no projeto de exemplo a utilização do MyProjectMigrator é bastante simples e pode ser customizada, mas se preferir outras maneiras de rodar as migrations, dê uma conferida na d<a href="https://github.com/schambers/fluentmigrator/wiki/Migration-Runners" target="_blank">documentação oficial</a> sobre Migration Runners.

Após a execução das migrations, como pode ser verificado nas imagens abaixo, o FluentMigrator irá criar uma tabela `VersionInfo` no seu banco de dados onde salvará a identificação das migrations que foram executadas, com a data e hora da execução das mesmas bem como sua descrição, que nada mais é do que o nome das classes.

{% img /images/manutencao-de-schema-de-banco-de-dados-com-migrations-em-net/database.png %}

{% img /images/manutencao-de-schema-de-banco-de-dados-com-migrations-em-net/version-info.png %}

#Conclusão

O que gosto nesta abordagem é a facilidade que qualquer desenvolvedor têm de realizar uma alteração no banco de dados, bem como compreender o que determinada migration, criada por outro desenvolvedor, irá realizar. A maneira que o FluentMigrator controla quais migrations foram executadas pelo número identificador, traz mais segurança na hora de atualizar o banco de dados de um cliente, e tudo isso permite que o processo de Integração Contínua funcione de maneira adequada. Ao invés de termos a necessidade de alguém executar algum software que gere um script de alteração de banco de dados, de maneira manual, a cada nova versão do sistema, basta fazer com que um Gerenciador ou Atualizar de Banco de Dados do projeto seja compilado com a versão correta do projeto que contêm as migrations.

Apesar de minha preferência pela utilização de migrations, é importante que você estude e adote a estratégia mais importante para o seu projeto. Se sua opção for utilizar migrations, não fique preso na maneira que eu utilizo, adapte para que funcione melhor para o seu projeto. Enfim, o que gostaria de passar neste post era isso, por favor deixe seu comentário com críticas, dúvidas ou sugestões.