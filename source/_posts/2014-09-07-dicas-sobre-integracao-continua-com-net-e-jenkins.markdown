---
layout: post
title: "Dicas sobre Integração Contínua com .NET e Jenkins"
date: 2014-09-07 16:10:00 -0300
comments: true
categories: ci, .net, jenkins
---

No momento em que escrevo este post estou trabalhando em uma aplicação _desktop_ para Windows desenvolvida na plataforma .NET. Com a primeira versão que será entregue para os usuários finais quase pronta, nossa equipe sentiu necessidade de colocar em prática o processo de Integração Contínua. Para atingir tal objetivo precisaríamos configurar uma ferramenta que nos auxiliasse a automatizar esse processo, e para isso escolhemos um servidor de Integração Contínua chamado <a href="http://jenkins-ci.org/" target="_blank">Jenkins</a>.

Não tenho a intenção de me aprofundar muito nos processos relacionados a Integração Contínua neste post, a ideia é documentar algumas dicas que encontrei durante a configuração do Jenkins para o nosso cenário atual. Espero que possa ser útil para mais pessoas, boa leitura. ;) 

<!--more-->

#WTF is Continous Integration?

Basicamente, Integração Contínua ou _Continous Integration_ (CI) é uma prática de desenvolvimento de software considerada por muitos um dos pilares das metodologias ágeis. Ela permite que a equipe encontre e elimine problemas rapidamente, agilizando os processos de _build_ e _delivery_, melhorando as respostas à falhas e aumentando a qualidade do software.

De maneira simplificada, um processo de integração continua deve monitorar as mudanças de código no seu repositório de controle de versão e realizar os passos necessários para gerar um executável entregável para o cliente. O que normalmente consiste em compilar a aplicação em modo _Release_, rodar os testes e gerar o executável.

Apesar de no nosso caso termos implementado um processo de Integração Contínua automatizado em um estágio um pouco mais avançado do ciclo de vida do projeto, é muito recomendável adotar essa prática já no início do desenvolvimento, obtendo assim os benefícios da Integração Contínua o mais rápido possível. 

#The Jenkins

O Jenkins é um servidor de Integração Contínua _open source_, multiplataforma e que pode ser utilizado para automatizar os processos de _build_ e _delivery_ de muitas linguagens, como C#, Ruby, PHP, Java, dentre outras. Um dos pontos principais do Jenkins são os seus inúmeros _plugins_, que estendem suas funcionalidades e podem ser desenvolvidos por qualquer um. Além disso, seu processo de instalação e configuração é relativamente simples.

#Qual processo queríamos automatizar?

Como citado anteriormente, estou trabalhando em uma aplicação _desktop_ para Windows desenvolvida na plataforma .NET, o processo de integração que precisávamos automatizar consistia basicamente nos seguintes passos:

1. Rebuild da aplicação em _Release_.
2. Execução do processo de _Code Analysis_.
3. Execução dos testes unitários.
4. Criação dos _setups_.

#Preparação do ambiente

A instalação do Jenkins no Windows é bem simples, basta baixar o instalador no <a href="http://jenkins-ci.org/" target="_blank">site oficial</a>, executar o mesmo e seguir o passo a passo. Ao final do processo o Jenkins estará instalado como um serviço e será executado automaticamente na inicialização do Windows. Para verificar se tudo ocorreu corretamente acesse o endereço <a href="http://localhost:8080/" target="_blank">localhost:8080</a> e veja se o Jenkins está rodando.

Uma dica importante para evitar dores de cabeça é forçar o serviço do Jenkins a ser executado com um usuário administrador do Windows ao invés de _Local System_ que é o padrão. No meu caso específico o Inno Setup não funcionou corretamente como _Local System_, para evitar esse tipo de problema execute os seguintes passos:

1. Abra o **Services** do Windows.
2. Abra as propriedades do serviço do **Jenkins**.
3. Clique na aba **Log On**.
4. Marque a opção **This account**.
5. Selecione um usuário administrador e digite a senha.
6. Clique em **OK** e reinicie o serviço.

{% img /images/dicas-sobre-integracao-continua-com-net-e-jenkins/jenkins-service.png %}

Além do Jenkins você precisará que todos os _softwares_ necessários para seus processos estejam instalados, como por exemplo o MSBuild, o NUnit, o Inno Setup, etc. Como esses _softwares_ são normalmente executados pelo Jenkins através do Command Prompt do Windows é interessante adicioná-los as variáveis de ambiente para que não seja necessário passar o caminho completo do executável, para isso siga os seguintes passos:

1. Abra o **System Properties** do Windows.
2. Clique na aba **Advanced**.
3. Clique no botão **Environment Variables**.
4. Em **System Variables** selecione a variável **Path**.
5. Clique em **Edit** e ao final da linha adicione os caminhos dos diretórios dos executáveis separados por ponto e vírgula.

{% img /images/dicas-sobre-integracao-continua-com-net-e-jenkins/environment-variables.png %}

#Estendendo o Jenkins

Antes de criarmos nosso primeiro _job_ podemos instalar alguns _plugins_ interessantes, para isso acesse o endereço <a href="http://localhost:8080/pluginManager/" target="_blank">localhost:8080/pluginManager</a> do Jenkins, esta página é bem simples e intuitiva, para instalar novos _plugins_ vá para a aba **Available**, para removê-los vá para a aba **Installed** e para atualizá-los vá para a aba **Updates**.

{% img /images/dicas-sobre-integracao-continua-com-net-e-jenkins/jenkins-manage-plugins.png %}

Logo abaixo segue uma lista de alguns dos _plugins_ que achei interessante e que utilizamos para esse nosso projeto. Existem centenas de outros _plugins_, fique a vontade para procurar novos.

* **BitBucket Plugin** - Permite acessar um repositório no BitBucket e configurar para um _job) ser iniciado quando um _Push_ for realizado no repositório.
* **MSBuild** - Permite realizar o _build_ de _projects_ e _solutions_ do Visual Studio.
* **NUnit** - Permite que o Jenkins utilize os relatórios gerados pelo NUnit.
* **Disk Usage Plugin** - Exibe estatísticas sobre o uso do disco.
* **Timestamper** - Adiciona o horário antes de cada linha do console do Jenkins, bastante útil para medir o tempo de realização de cada tarefa.
* **Google Cloud Messaging Notification Plugin** - Envia notificações para _smartphones_ Android.
* **Safe Restart Plugin** - Permite realizar o _restart_ do Jenkins de maneira segura.

#Segurança

Uma dica que considero importante, principalmente se você vai permitir acesso externo ao seu servidor Jenkins, é desativar a funcionalidade de _Sign up_ e controlar as permissões dos usuários, para isso siga os seguintes passos:

1. Acesse o endereço <a href="http://localhost:8080/configureSecurity/" target="_blank">localhost:8080/configureSecurity</a>.
2. Marque a opção **Enable security**.
3. Em **Security Realm** selecione a opção **Jenkins’ own user database** e desmarque a opção **Allow users to sign up**.
4. Em **Authorization** selecione a opção **Matrix-based security**, adicione os usuários e marque as opções que deseja que o usuário tenha permissão.

{% img /images/dicas-sobre-integracao-continua-com-net-e-jenkins/jenkins-security.png %}

Como desativamos a permissão de _Sign up_ você precisará criar as contas dos usuários, para isso acesse o endereço <a href="http://localhost:8080/securityRealm/" target="_blank">localhost:8080/securityRealm</a>. Um detalhe importante aqui é que o **Username** deve ser o mesmo que foi adicionado no passo anterior para que as permissões funcionem corretamente.

#Configurações

As configurações do Jenkins podem ser acessadas através do endereço <a href="http://localhost:8080/configure/" target="_blank">localhost:8080/configure</a>, dentre outras coisas, aqui você pode configurar o servidor SMTP de envio de e-mail, o local onde seu Git está instalado (que normalmente é reconhecido automaticamente), as configurações para o Google  Cloud Messaging Notification, etc.

#Criando seu primeiro job

Para criar um novo _job_ no Jenkins, acesse o endereço <a href="http://localhost:8080/newJob" target="_blank">localhost:8080/newJob</a>, digite um nome para o _job_, marque a opção **Freestyle project** e clique em **OK**. Você será levado para a página de configuração do seu _job_ recém criado, está página é muito importante pois é aqui que devemos configurar tudo que o nosso _job_ irá realizar.

##Custom workspace

A primeira dica aqui é alterar o _workspace_ onde o _job_ será realizado, por padrão ele fica dentro do diretório onde o Jenkins foi instalado, que normalmente é em `C:\Program Files (x86)\Jenkins`, o que pode desencadear alguns erros relacionados a permissão de escrita se algum dos _softwares_ utilizados pelo seu job tentar criar algum arquivo em disco. Para alterar a _workspace_ clique no botão **Advanced** em  **Advanced Project Options**, marque a opção **Use custom workspace** e informe o caminho do diretório, por exemplo: `C:\CI\my-job\workspace`.

##Git e BitBucket

Caso tenha instalado o _plugin_ do BitBucket (e consequentemente o do Git), em **Source Code Managment** selecione a opção **Git** e configure seu repositório, sua _branch_ e suas credenciais de acesso. Um dica importante aqui é alterar o _timeout_ de _clone_ e _fetch_ do Git, o padrão é de apenas 10 minutos, o que pode não ser suficiente dependendo do tamanho do seu repositório  e da velocidade de sua conexão. Até descobrir isso perdi alguns minutos, para você não passar por isso também, clique no botão **Add**, escolha a opção **Advanced Clone Behaviors** e informe um tempo que ache razoável para o _timeout_.

Com o _plugin_ do BitBucket você pode configurar para que seu job ser iniciado assim que um push no repositório seja feito, para isso marque a opção **Build when change is pushed to BitBucket**. Para essa opção funcionar você precisará adicionar um _Hook_ de _POST_ apontando para <a href="http://seu-ip-publico:8080/bitbucket-hook/" target="_blank">seu-ip-publico:8080/bitbucket-hook</a> nas configurações do seu repositório no BitBucket.

##MSBuild

Para configurar o _build_ do seu projeto no MSBuild clique em **Add build step** e na opção **Build a Visual Studio project or solution using MSBuild**. Em **MSBuild Build File** informe o caminho da _solution_ no _workspace_ do _job_, por exemplo: `C:\CI\my-job\workspace\MyProject.sln`. Em **Command Line Arguments** você deverá informar os parâmetros do seu _build_, por exemplo: `/t:Rebuild /p:Configuration="Release" /p:Platform=x86 /p:DevEnvDir="C:\Program Files (x86)\Microsoft Visual Studio 12.0\Common7\IDE"`. O parâmetro `DevEnvDir` é importante, ele é utilizado para algumas coisas durante o processo de _build_, como por exemplo na execução do _Code Analisys_. Clicando em **Advanced** você também pode marcar a opção **If warning set the build to Unstable**.

##NUnit

Os testes do NUnit serão executados através do Console, para configurá-lo clique em **Add build step** e na opção **Execute Windows batch command**, no campo **Command** você deverá informar o nome do executável do NUnit (ou o caminho completo caso não o tenha adicionado nas variáveis de ambiente do Windows), o caminho do teste no _workspace_ e o nome do arquivo XML onde será salvo o relatório com os resultados dos testes. Por exemplo: `"nunit-console-x86.exe" "MyTest\bin\x86\Release\MyTest.dll" /xml=MyTest.dll-nunit-result.xml`.

Para que o Jenkins exiba os resultados dos testes clique em **Add post-build action** e na opção **Publish NUnit test result report** e em **Test report XMLs** e informe `*nunit-result.xml`.

##Inno Setup

Para gerar os executáveis utilizando o Inno Setup clique novamente em **Add build step** e na opção **Execute Windows batch command**, no campo **Command** informe o nome do executável do Inno Setup (ou o caminho completo caso não o tenha adicionado nas variáveis de ambiente do Windows) e o caminho do seu _script_ no _workspace_. Por exemplo: `"Compil32.exe" /cc "setup.iss"`.

#Finalizando

Como comentei no início do _post_, não pretendia me aprofundar nos conceitos relacionados a Integração Contínua e sim compartilhar algumas dicas que acredito serem úteis. A intenção também não era ser nenhuma receita de bolo, adapte o que for necessário para a realidade do seu projeto, o Jenkins é uma ferramenta muito poderosa, explore seus recursos.

Espero que tenha gostado do _post_, por favor deixe críticas, dúvidas ou sugestões nos comentários.