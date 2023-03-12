<h1 align="center">
  <p align="center">Projeto: hospedando um servidor de Minecraft semi-automatizado na Amazon Web Services (AWS)</p>
  <img src="https://i.imgur.com/ZKdQQfb.png" alt="Banner">
</h1>

Concluí recentemente a trilha de Cloud do programa Start by Capgemini. Uma lição importante que a minha graduação em Letras: Português - Francês me trouxe foi: só garantimos que algo foi aprendido se, para além de conseguir colocar em prática, conseguimos explicar.

Decidi então escrever esse artigo como forma de tutorial para explicar como consegui hospedar um servidor de Minecraft usando a Amazon Web Services.

O fator que me motivou foi a minha infância, onde possuí um servidor de Minecraft com amigos usando uma plataforma como um serviço (PaaS). A meta agora era utilizar uma infraestrutura como serviço (IaaS).

## Características do servidor
* Suporte até 20 jogadores simultâneos;
* Backup automático do servidor toda vez que a instância (EC2) for parada;
* O servidor é desligado automaticamente quando não há jogadores conectados, reduzindo custos;
* Inicialização da instância e do servidor através de um usuário do IAM disponibilizado para os amigos.

## Serviços e ferramentas da Amazon Web Services que serão utilizados
* ``Amazon Systems Manager``;
* ``Amazon Elastic Compute Cloud (EC2)``;
* ``Amazon Simple Storage Service (S3)``;
* ``Amazon CloudWatch``;
* ``Amazon EventBridge``;
* ``Amazon Identity and Access Management (IAM)``;
* ``Amazon Command Line Interface (CLI)``.

## Requisitos
* Conta na AWS;
* Conhecimentos básicos de Linux.

## Custos
Usando como exemplo 60 horas jogadas por mês, o servidor custaria em torno de 18-20 reais por mês. O preço pode variar para mais ou para menos, a depender da cotação do dólar e do tipo de instância escolhida. O servidor também pode sair de graça se a instância escolhida estiver no ["Free Tier"](https://aws.amazon.com/pt/free/).

# Instalação
## 1. Reservando um "terreno" na nuvem
Logue na sua conta da AWS e verifique no canto superior direito se na região consta "São Paulo". Isso garantirá que seu servidor ficará hospedado no Brasil. Em seguida, procure por VPC. Após acessar o serviço, você será conduzido a um painel. Clique em "Criar VPC".

Para diminuir as etapas do processo e agilizar, escolha a opção "VPC e muito mais". Assim, nessa única tela, conseguiremos: reservar um terreno (VPC), criaremos um lote (sub-rede) dentro desse terreno, construiremos uma casa nesse lote (instância do EC2 onde o Minecraft irá funcionar) e colocaremos um portão aberto (Gateway da Internet) no nosso terreno. Para que as pessoas possam entrar no lote onde está a nossa casa, um caminho (tabela de rotas pública) será criado entre o portão aberto do terreno e o nosso lote.

Configuração:

* Geração automática da etiqueta de nome > gerar automaticamente > digite "minecraft". Isso acrescentará "minecraft" nas etiquetas para que possamos identificar tudo que esteja relacionado ao servidor;
* Bloco CIDR IPV4: 10.0.0.0/16
* Bloco CIDR IPv6: nenhum;
* Locação: padrão;
* Número de zonas de disponibilidades (AZs): 1 (em Personalizar AZs, selecione "sa-east-1a");
* Número de sub-redes públicas: 1;
* Número de sub-redes privadas: 0;
* Personalizar blocos CIDR de sub-redes: 10.0.1.0/24
* Gateways NAT (USD): nenhuma;
* Endpoints da VPC: nenhuma;
* Opções de DNS: marque as duas opções "Habilitar nomes de host DNS" e "Habilitar resolução de DNS";
* Clique em "Criar VPC".

![Imagem da configuração](https://media.licdn.com/dms/image/D4D12AQFyT4NbR9XHcw/article-inline_image-shrink_1000_1488/0/1673906437473?e=1684368000&v=beta&t=7IcErcty4K_h9k1ErzxI_cGdiGFxg5AkGrvvlpdyGNk)

## 2. Gerenciando permissões
### 2.1 Criando políticas
Iremos gerenciar as permissões do que pode ser feito por pessoas/instâncias, quais recursos elas podem acessar. Deixaremos o nosso "background" pronto.
#### 2.1.1 Política para a instância
Permitirá que a nossa instância use o AWS CLI para salvar o backup do servidor no S3 e desligue a instância.
* Procure por IAM e em seguida clique em "Políticas" no menu do painel da IAM;
* Clique em "Criar política";
* Escolha a opção JSON e coloque o código abaixo;
```json
{   
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "ec2:StopInstances",
            "Resource": "*"
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject"
            ],
            "Resource": "arn:aws:s3:::*/*"
        }
    ]
}
```
* Clique em próximo sem alterar nada até "Revisar";
* O nome da policy fica a sua escolha, eu coloquei: policy-minecraft-server;
* Agora só clicar em "Criar Política" e pronto.

#### 2.1.2 Política para o jogador
Permitirá que o perfil de usuário a qual essa política estará associada tenha apenas a permissão de ligar o servidor. Os jogadores/amigos que possuírem o perfil usuário poderão então se logar no console de gerenciamento da AWS e ligar o servidor quando ele estiver desligado.
* Faça os mesmos procedimentos da política para a instância;
* Na parte da opção JSON, cole o código abaixo;
```json
{  
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances",
                "ec2:StopInstances",
                "ec2:DescribeInstanceStatus"
            ],
            "Resource": "*"
        }
    ]
}
```
* Na parte de revisar, o nome fica a sua escolha, coloquei: policy-minecraft-player

### 2.2 Criando usuário
O usuário permitirá que uma pessoa logue na sua conta da Amazon, mas só tenha a permissão de iniciar a(s) instância(s) devido a policy que criamos anteriormente e que será anexada.
* Volte ao painel do IAM e escolha a opção "Usuários";
* Clique em "Adicionar usuários";
* Nome de usuário: minecraft-player;
* Tipo de credencial da AWS: selecione apenas "Senha: acesso ao Console de Gerenciamento da AWS";
* Senha do console > senha personalizada > digite a senha que preferir;
* Desmarque a opção "Exigir redefinição de senha" e clique em próximo;
* Em definir permissões, selecione: "Anexar políticas existentes de forma direta" e escolha a política que criamos anteriormente "policy-minecraft-player" e clique em próximo até chegar a opção "Criar usuário";
* Ao criar o usuário, uma mensagem aparecerá informando como logar no usuário que você criou usando as credenciais criadas.

### 2.3 Criando função (role)
A função funciona quase da mesma forma que um usuário. Entretanto, ela é assumida por serviços. Por exemplo, graças a policy que criamos vinculada a função que criaremos, a nossa instância do EC2 poderá "vestir" a função e gravar um arquivo em um bucket do S3.
* Volte ao painel do IAM e escolha a opção "Funções";
* Clique em "Criar função";
* Em tipo de entidade confiável, selecione "Serviço da AWS";
* Em casos de uso comuns, selecione "EC2" e clique em "Próximo";
* Selecione a política de permissões que criamos anteriormente "policy-minecraft-server" **e também uma já gerenciada chamada "AmazonSSMManagedInstanceCore"**. Essa última permitirá a conexão na instância através do Session Manager do Systems Manager, sem precisar expor a porta 22 para realizar conexão SSH. Além de permitir também que o Systems Manager envie comandos para a instância (precisaremos disso futuramente). Clique em próximo.
* Em nome da função, coloque "minecraft-role" e clique em "Criar função".

### 2.4 Criando um grupo de segurança
O grupo de segurança permitirá que os jogadores se conectem no servidor de Minecraft. Procure pelo serviço EC2 e escolha no painel a opção "Security Groups" na seção "Rede e segurança". Em seguida, clique em "Criar grupo de segurança".
* Nome do grupo de segurança: minecraft-security-group;
* Descrição: liberar portas para se conectar no Minecraft;
* Em VPC, certifique que consta a VPC que criamos anteriormente;
* Regras de entrada: clique em adicionar regra;
* Em tipo, escolha: TCP personalizado;
* Intervalo de portas: 25565 (é a porta que permite a conexão com o servidor do Minecraft);
* Origem: 0.0.0.0/0 (permitirá que qualquer pessoa se conecte);
* Em regras de saída, deixe padrão e clique em "Criar grupo de segurança".

![Imagem da configuração](https://media.licdn.com/dms/image/D4D12AQHQtC7zIP8cLQ/article-inline_image-shrink_1000_1488/0/1674000993538?e=1684368000&v=beta&t=Hfi6mevhzxe4x1nt6HfkPPFfSRbj1DG07cQzV71C4fg)

## 3. Provisionando um bucket do S3
O S3 ficará responsável por armazenar os backups do nosso servidor.
## 3.1 Criando o bucket
Procure por S3 e clique em "Criar bucket".
* Nome do bucket (precisa ser um nome único global): ramonrosa-minecraft-backup (exemplo);
* Região da AWS: América do Sul (São Paulo) sa-east-1;
* Propriedade de objeto: ACLs desabilitadas;
* Configurações de bloqueio do acesso público deste bucket: marque a opção "bloquear todo o acesso público";
* Versionamento de bucket: selecione "Ativar". O versionamento permitirá o armazenamento de várias versões do mesmo arquivo. Isso será benéfico, já que uparemos o backup sempre com o mesmo nome;
* Em criptografia padrão, deixe como está;
* Clique em "Criar bucket";

## 3.2 Regras de ciclo de vida
Para evitar que todos os backups fiquem salvos, consumindo muito espaço, criaremos uma regra de ciclo de vida pra excluir os backups após 7 dias, mantendo sempre os últimos 7 backups.

Volte ao painel do Amazon S3, o bucket que você acabou de criar já deve aparecer. Selecione ele e navegue até a aba "Gerenciamento". Em seguida, clique na opção: "Criar regra de ciclo de vida".

![Imagem da configuração](https://media.licdn.com/dms/image/D4D12AQHcZ0bMkn3IqA/article-inline_image-shrink_1500_2232/0/1674003339243?e=1684368000&v=beta&t=4F6W3vFCXPTOsOWxfdWiBK-M9SO7gq-Bm2L89fnJkgc)

* Nome da regra de ciclo de vida: excluir-backups-antigos;
* Escolha um escopo de regra: aplicar a todos os objetos no bucket;
* Ações de regras de ciclo de vida: selecione a opção "Excluir permanentemente versões desatualizadas de objetos";
* Dias após os objetos ficarem desatualizados: 7;
* Número de versões mais recentes a serem retidas: 7;
* Clique em "Criar regra";

![Imagem da configuração](https://media.licdn.com/dms/image/D4D12AQGegWEzl2AMrQ/article-inline_image-shrink_1500_2232/0/1674004377950?e=1684368000&v=beta&t=zZhGeWRcplKL1RPeQXQeW7Y7RsHfylg21ypchY7OgX4)

## 4. Provisionando a instância do EC2
A instância será a "máquina" onde o servidor de Minecraft ficará ligado.

### 4.1 Criando a instância 
Nessa etapa escolheremos o hardware necessário para o nosso servidor.
Acesse o painel do EC2, clique em "Instâncias" e em seguida "Executar instância":
* Nome: servidor-minecraft;
* Imagens de aplicação e de sistema operacial: Amazon Linux 2 | Arquitetura: 64 bits (Arm);
* Tipo de instância: m6g.medium (o tipo de instância é a "potência" da nossa instância. O Minecraft faz parte de um grupo de servidores de jogos que os seus processos principais rodam de forma single thread, então 1vCPU Graviton2 já deve ser o suficiente. Optei por 4gb de ram, pois permite futuramente adicionar alguns mods e comportar mais jogadores. Caso tenha dúvida sobre qual instância escolher, só olhar a descrição na documentação da amazon);
* Key pair: gere um par de chaves (ela é usada pra acessar a instância via SSH, caso a porta 22 esteja liberada no grupo de segurança). No nosso caso não utilizaremos SSH, não deixaremos a porta 22 exposta, iremos acessar a instância via Systems Manager);

![Imagem da configuração](https://media.licdn.com/dms/image/D4D12AQFfdnQ5mgbE-g/article-inline_image-shrink_1000_1488/0/1674086580470?e=1684368000&v=beta&t=MubGiMRaXxzuEBA_s8fO2MkGY1XhVcKL1CVcjsSLdAU)

* Configurações de rede: clique em editar.
* Verifique se a VPC e a sub-rede criadas anteriormente estão selecionadas;
* Atribuir IP público automaticamente: Habilitar;
* Firewall (grupos de segurança): clique em selecionar grupo de segurança existente e escolha "minecraft-security-group";

![Imagem da configuração](https://media.licdn.com/dms/image/D4D12AQE80XjBi7F2dg/article-inline_image-shrink_1500_2232/0/1674086835767?e=1684368000&v=beta&t=HBok18V4Rtm1a3Xo3gv98dxKGWInAX05Knuw99JkHjA)

* Configurar armazenamento: pode deixar o padrão (8 Gib gp2 Root volume);
* Clique em detalhes avançados;
* Perfil de instância do IAM: minecraft-role (o perfil que criamos anteriormente);
* Dados do usuário: são os comandos que a instância irá executar apenas a primeira vez que for inicializada. Cole o código abaixo:

```bash
#!/bin/bash
yum update -y
rpm --import https://yum.corretto.aws/corretto.key 
curl -L -o /etc/yum.repos.d/corretto.repo https://yum.corretto.aws/corretto.repo
yum install -y java-19-amazon-corretto-devel
adduser minecraft
mkdir /opt/minecraft
mkdir /opt/minecraft/server
mkdir /opt/minecraft/backup
wget -P /opt/minecraft/server https://piston-data.mojang.com/v1/objects/c9df48efed58511cdd0213c56b9013a7b5c9ac1f/server.jar
echo "eula=true" > /opt/minecraft/server/eula.txt
chown -R minecraft:minecraft /opt/minecraft
```

(O código acima irá: atualizar os pacotes, instalar o "java da Amazon", criar o usuário "minecraft", criar os diretórios para a organização do servidor, baixar o arquivo principal do Minecraft no diretório /opt/minecraft/server, criar o arquivo "eula.txt" no mesmo diretório para o servidor ser inicializado e finalmente colocar o usuário "minecraft" como proprietário de todos esses diretórios e arquivos criados e baixados)

* Revise as informações em "Resumo" e clique em "Executar instância";
* A instância será inicializada. Para visualizar os detalhes da sua instância, só acessar o painel EC2 e conferir em "Instâncias". Selecione a instância e você verá os detalhes. O IP que você utilizará no Minecraft para se conectar no servidor consta em "Endereço IPv4 público".

### 4.2 Providenciando um IP Elástico pra instância
Se não provisionarmos um IP Elástico pra nossa instância, toda vez que ela for desligada/ligada, o IP público irá mudar, esse não é um cenário ideal. Vamos então alocar um IP Elástico e anexá-lo a instância.
* No painel EC2, procure por "IPs elásticos" em "Rede e segurança" e clique em "Alocar endereço IP elástico";
* Deixe tudo padrão e clique em "Alocar";
* O IP elástico será alocado e você já poderá conferi-lo no menu para onde você é conduzido após alocar. A numeração em "Endereço IPv4 alocado" é o que você utilizará para se conectar no servidor;
* Agora vamos associar esse IP elástico a nossa instância EC2;
* Selecione o endereço IPv4 e clique em "Ações" (ao lado de Alocar endereço IP elástico) > "Associar endereço IP elástico";
* Em tipo tipo de recurso: selecione instância;
* Em instância: selecione a instância que criamos no EC2;
* Por fim, clique em "Associar";

### 4.3 Acessando a instância EC2
Utilizaremos o Systems Manager para nos conectarmos a instância. Não faremos conexão SSH, pois não deixamos a porta 22 exposta. Para isso:
* Procure por "Systems Manager" e no painel procure por "Gerenciador de sessão" na categoria "Gerenciamento de nós";
* Clique em "Iniciar sessão" e em "Instâncias de destino", escolha a instância EC2 que nós criamos anteriormente. Confirme novamente em "Iniciar sessão";

### 4.4 Iniciando o servidor de Minecraft
Já conectados a instância, poderíamos simplesmente digitar o comando abaixo para iniciar o servidor:

```bash
cd /opt/minecraft/server
java -Xmx4096M -Xms4096M -jar server.jar nogui
```

Entretanto, fazer isso não é o ideal, pois teríamos que executar esse comando toda vez que a nossa "máquina" (instância) fosse desligada e ligada. Além disso, o servidor de Minecraft possui um console próprio, e precisamos acessar esse console toda vez que quisermos enviar um comando dentro do servidor.

Para resolver esses problemas, faremos uso de duas ferramentas que já estão instaladas no Amazon Linux 2:
* Criaremos um serviço com o "SystemD". Dessa forma, o servidor de Minecraft inicializará automaticamente toda vez que a instância for ligada;
* Faremos uso do pacote "Screen" dentro do script do serviço criado. Graças a esse pacote, conseguiremos alternar as telas entre o Shell e o console do Minecraft.

Vamos agora colocar em prática:
* Vamos criar o nosso serviço "minecraft.service" no diretório /etc/systemd/system usando os comandos:

```bash
cd /etc/systemd/system
sudo vi minecraft.service
```

* Uma tela irá abrir para edição do conteúdo do nosso minecraft.service. Aperte a tecla "INSERT" até aparecer --INSERT-- no canto inferior esquerdo. Em seguida, cole o código abaixo:

```service
[Unit]
Description=Minecraft Server
After=network.target
[Service]
WorkingDirectory=/opt/minecraft/server
Type=simple
PrivateUsers=true
User=minecraft
Group=minecraft
ProtectSystem=full
ProtectHome=true
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true
ExecStart=/usr/bin/screen -DmS consolemine /usr/bin/java -Xmx4096M -Xms4096M -jar server.jar nogui
ExecStop=/usr/bin/screen -S consolemine -X stuff "save-all^M"
ExecStop=/usr/bin/screen -S consolemine -X stuff "stop^M"
Restart=on-failure
RestartSec=30s
[Install]
WantedBy=multi-user.target
Restart=on-failure
RestartSec=30s
[Install]
WantedBy=multi-user.target
```

* Aperte a tecla ESC digite ":" para que a caixa de texto embaixo seja liberada. Após, digite "wq" e aperte ENTER;
* Digite os comandos abaixo para ativar o nosso serviço que foi criado, e também colocá-lo na inicialização do sistema. Dessa forma, toda vez que a "máquina" for ligada, o servidor também será iniciado. Se por algum acaso o servidor "crashar", parar de responder, ele também será reiniciado automaticamente:

```bash
sudo systemctl daemon-reload
sudo systemctl enable minecraft
sudo systemctl start minecraft
```

* Pronto, o servidor está funcionando e agora você pode se conectar pelo Minecraft usando o IP Elástico que alocamos e associamos anteriormente.
* Caso você queira acessar o console do Minecraft, basta mudar para o user Minecraft e ver o ID da sessão do screen :

```bash
sudo su minecraft
screen -ls
```

* Em seguida, digite:

```bash
screen -r IDdasessao
```

* Para sair do console do minecraft e voltar pro Shell, só pressionar CTRL + A + D;

## 5. Backup do servidor e desligamento da instância automáticos
Para essa etapa, precisaremos de quatro serviços e uma ferramenta:
* Amazon S3 (já criamos anteriormente);
* Amazon CloudWatch;
* Amazon EventBridge;
* Amazon Systems Manager;
* AWS CLI

Normalmente, quando há jogadores no servidor, o uso da CPU se mantém elevado. Iremos utilizar o CloudWatch para monitorar o uso da CPU a cada intervalo de tempo. Criaremos um alarme no CloudWatch para ser ativado assim que o uso da CPU atingir uma porcentagem mínima estabelecida. Ao ativar o alarme, o EventBridge pedirá ao Systems Manager para enviar um comando para a nossa instância para que o servidor seja fechado, um backup do servidor seja feito e enviado para o bucket do S3 e finalmente para que nossa instância seja desligada.

## 5.1 Criando o alarme
* Procure por CloudWatch, procure por "Alarmes" no painel, selecione "Todos os alarmes" e clique em "Criar alarme";
* Em "Selecionar métrica", procure pelo ID da sua instância EC2 (é possível obter no painel do EC2). Selecione a opção "CPUUtilization" e em seguida "Selecionar métrica";

![Imagem da configuração](https://media.licdn.com/dms/image/D4D12AQF_NvPbWeMeLw/article-inline_image-shrink_1500_2232/0/1674104868910?e=1684368000&v=beta&t=_CEnEnBR1opZsu7EMTKTPW-6fwQcwgQA_v4np7q0D30)

* Em estatística, coloque "Média" e período "5 minutos";
* Tipo de limite: Estático;
* Sempre que CPUUtilization for "Menor" que "6";
* Pontos de dados para alarme: 1 de 1;
* Clique em próximo;

![Imagem da configuração](https://media.licdn.com/dms/image/D4D12AQF6B-KRn4RadA/article-inline_image-shrink_1500_2232/0/1674106205909?e=1684368000&v=beta&t=4lu2KKnxH9-XC7MD9-w0OYzgpuB072Qp2CESRZKsCUk)

* Em configurar ações, se em notificação houver alguma, clique em "Remover" e em seguida "Próximo";
* Nome do alarme: "jogadores-nao-conectados" e em seguida clique em "Criar alarme";
* O alarme será criado, porém aparecerá "Dados insuficientes". É normal, é necessário esperar os dados serem coletados na métrica.

## 5.2 Efetuando o backup e desligando a instância pelo EventBridge
* Procure pelo serviço Amazon EventBridge, no seção "Ônibus" do menu, clique em "Regras". Em seguida clique em "Criar regra";
* Nome da regra: "backup-and-stop";
* Tipo de regra: "Regra com um padrão de eventos" e clique em próximo;
* Fonte do evento: "Eventos da AWS ou de parceiros do EventBridge";
* Pule a seção "Evento de exemplo - opcional";
* Método: "Usar formulário de padrão";
* Fonte do evento: "Serviços da AWS";
* Serviço da AWS: "CloudWatch";
* Tipo de evento: "CloudWatch Alarm State Change" e em seguida clique em "Próximo";
* Tipos de destino: "Serviço da AWS"
* Selecionar um destino: "Comando de execução do Systems Manager";
* Documento: "AWS-RunShellScript";
* Chave de destino: digite exatamente "InstanceIds";
* Valor(es) de destino: digite aqui o ID da sua instância disponível no painel do EC2 e clique em "Adicionar";
* Configurar parâmetro(s) de automação: "Constante";
* Adicione cada um dos comandos abaixo separadamente. Eles irão fechar o servidor salvando todos os arquivos ao fechar. Em seguida os arquivos do servidor serão "zipados" na pasta backup e em seguida esses arquivos zipados serão transferidos para o nosso bucket do S3. Finalmente a instância será desligada.

```bash
sudo systemctl stop minecraft

sudo zip -r /opt/minecraft/backup/backupminecraft.zip /opt/minecraft/server

aws s3 mv /opt/minecraft/backup/backupminecraft.zip s3://DigiteAquiONomeDoSeuBucketS3

aws ec2 stop-instances --instance-ids digiteaquioIDdaInstancia --region sa-east-1
```

* Perfil de execução: "Criar um novo perfil para este recurso específico" e clique em "Próximo" até chegar em "Criar regra";

# Conclusão
O servidor está pronto para ser utilizado. Lembrando que para iniciar o servidor, basta fornecer as credenciais para os seus amigos do usuário do IAM que criamos lá no início com as permissões apenas de iniciar a instância. Eles irão acessar o console de gerenciamento da AWS e iniciar a instância lá. Não precisa se preocupar de mexerem na sua conta ou criarem algum serviço, acarretando em uma fatura alta no final do mês. Se você seguiu o tutorial, eles só terão a permissão de iniciar a instância já existente. A instância desligará automaticamente caso não haja players jogando, ajudando a reduzir a fatura no final do mês. Em relação aos backups, eles estarão disponível no seu bucket do S3, junto a versão e a data/hora de última modificação.

# Alternativas de implementação
Nas referências você pode encontrar outras formas de implementação, como por exemplo usar o Amazon SNS para ativar o servidor via email. Ou então usar o Amazon 53 juntamente com o stream de logs do CloudWatch para iniciar o servidor assim que um jogador tentar se conectar.

# Referências 
AWS Documentation. Disponível em: https://docs.aws.amazon.com/

BRAS, Julien. **How to Run a Minecraft Server on AWS For Less Than 3 US$ a Month**. Disponível em: https://dev.to/julbrs/how-to-run-a-minecraft-server-on-aws-for-less-than-3-us-a-month-409p

GIBSON, Ray. **Minecraft on-demand**. Disponível em: https://github.com/doctorray117/minecraft-ondemand

KRIEGL, Frank. **Setting up an on-demand Minecraft server on AWS**. Disponível em: https://franok.de/techblog/2022/on-demand-minecraft-server-on-aws.html

MCKENZIE, Keran. **Setup Minecraft Server (java edition) in AWS EC2**. Disponível em: https://www.linkedin.com/pulse/setup-minecraft-server-java-edition-aws-ec2-keran-mckenzie/

**Tutorials/Server startup script**. Minecraft Wiki. Disponível em: https://minecraft.fandom.com/wiki/Tutorials/Server_startup_script

Screen User's Manual. Disponível em: https://www.gnu.org/software/screen/manual/screen.html
