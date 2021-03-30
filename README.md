# Falhas de handshake SSL

# 1. Visão Geral
Secured Socket Layer (SSL) é um protocolo criptográfico que fornece segurança na comunicação pela rede. Neste tutorial, discutiremos vários cenários que podem resultar em uma falha de handshake SSL e como fazê-lo.

Observe que nossa Introdução ao SSL usando JSSE cobre os fundamentos do SSL com mais detalhes.

# 2. Terminologia
É importante observar que, devido a vulnerabilidades de segurança, o SSL como padrão é substituído pelo Transport Layer Security (TLS). A maioria das linguagens de programação, incluindo Java, tem bibliotecas para oferecer suporte a SSL e TLS.

Desde o início do SSL, muitos produtos e linguagens como OpenSSL e Java tinham referências ao SSL que mantiveram mesmo depois que o TLS assumiu. Por esse motivo, no restante deste tutorial, usaremos o termo SSL para nos referirmos geralmente a protocolos criptográficos.

# 3. Configuração
Para o propósito deste tutorial, criaremos um servidor simples e aplicativos cliente usando a API Java Socket para simular uma conexão de rede.

3.1. Criando um cliente e um servidor
Em Java, podemos usar sockets para estabelecer um canal de comunicação entre um servidor e um cliente na rede. Os soquetes fazem parte do Java Secure Socket Extension (JSSE) em Java.

Vamos começar definindo um servidor simples:

```
int port = 8443;
ServerSocketFactory factory = SSLServerSocketFactory.getDefault();
try (ServerSocket listener = factory.createServerSocket(port)) {
    SSLServerSocket sslListener = (SSLServerSocket) listener;
    sslListener.setNeedClientAuth(true);
    sslListener.setEnabledCipherSuites(
      new String[] { "TLS_DHE_DSS_WITH_AES_256_CBC_SHA256" });
    sslListener.setEnabledProtocols(
      new String[] { "TLSv1.2" });
    while (true) {
        try (Socket socket = sslListener.accept()) {
            PrintWriter out = new PrintWriter(socket.getOutputStream(), true);
            out.println("Hello World!");
        }
    }
}
```

O servidor definido acima retorna a mensagem “Hello World!” para um cliente conectado.

A seguir, vamos definir um cliente básico, que conectaremos ao nosso SimpleServer:

```
String host = "localhost";
int port = 8443;
SocketFactory factory = SSLSocketFactory.getDefault();
try (Socket connection = factory.createSocket(host, port)) {
    ((SSLSocket) connection).setEnabledCipherSuites(
      new String[] { "TLS_DHE_DSS_WITH_AES_256_CBC_SHA256" });
    ((SSLSocket) connection).setEnabledProtocols(
      new String[] { "TLSv1.2" });
    
    SSLParameters sslParams = new SSLParameters();
    sslParams.setEndpointIdentificationAlgorithm("HTTPS");
    ((SSLSocket) connection).setSSLParameters(sslParams);
    
    BufferedReader input = new BufferedReader(
      new InputStreamReader(connection.getInputStream()));
    return input.readLine();
}
```

Nosso cliente imprime a mensagem retornada pelo servidor.

### 3.2. Criação de certificados em Java
SSL fornece sigilo, integridade e autenticidade nas comunicações de rede. Os certificados desempenham um papel importante no que diz respeito ao estabelecimento de autenticidade.

Normalmente, esses certificados são adquiridos e assinados por uma autoridade de certificação, mas para este tutorial, usaremos certificados autoassinados.

Para conseguir isso, podemos usar o keytool, que vem com o JDK:

```
$ keytool -genkey -keypass password \
                  -storepass password \
                  -keystore serverkeystore.jks
```

O comando acima inicia um shell interativo para coletar informações para o certificado, como Nome Comum (CN) e Nome Distinto (DN). Quando fornecemos todos os detalhes relevantes, ele gera o arquivo serverkeystore.jks, que contém a chave privada do servidor e seu certificado público.

Observe que serverkeystore.jks é armazenado no formato Java Key Store (JKS), que é proprietário do Java. Atualmente, o keytool nos lembra que devemos considerar o uso de PKCS # 12, que também é compatível.

Podemos ainda usar o keytool para extrair o certificado público do arquivo keystore gerado:

```
$ keytool -export -storepass password \
                  -file server.cer \
                  -keystore serverkeystore.jks
```

O comando acima exporta o certificado público do armazenamento de chaves como um arquivo server.cer. Vamos usar o certificado exportado para o cliente, adicionando-o ao seu armazenamento confiável:

```
$ keytool -import -v -trustcacerts \
                     -file server.cer \
                     -keypass password \
                     -storepass password \
                     -keystore clienttruststore.jks
```

E mais detalhes sobre o uso do keystore do Java podem ser encontrados em nosso tutorial anterior.

# 4. SSL Handshake
Os handshakes SSL são um mecanismo pelo qual um cliente e um servidor estabelecem a confiança e a logística necessárias para proteger sua conexão pela rede.

Este é um procedimento muito orquestrado e compreender os detalhes disso pode ajudar a entender por que ele freqüentemente falha, o que pretendemos abordar na próxima seção.

As etapas típicas em um handshake SSL são:

- O cliente fornece uma lista de possíveis versões SSL e conjuntos de criptografia para uso;
- O servidor concorda com uma determinada versão SSL e conjunto de criptografia, respondendo de volta com seu certificado;
- O cliente extrai a chave pública do certificado e responde com uma “chave pré-mestre” criptografada;
- O servidor descriptografa a “chave pré-mestra” usando sua chave privada;
- Cliente e servidor calculam um “segredo compartilhado” usando a “chave pré-mestre” trocada;
- O cliente e o servidor trocam mensagens confirmando a criptografia e descriptografia bem-sucedidas usando o “segredo compartilhado”.

Embora a maioria das etapas sejam as mesmas para qualquer handshake SSL, há uma diferença sutil entre SSL unilateral e bidirecional. Vamos revisar rapidamente essas diferenças.

### 4.1. O handshake em SSL unilateral
Se nos referirmos às etapas mencionadas acima, a etapa dois menciona a troca de certificado. O SSL unilateral requer que um cliente possa confiar no servidor por meio de seu certificado público. Isso permite que o servidor confie em todos os clientes que solicitam uma conexão. Não há como um servidor solicitar e validar o certificado público de clientes, o que pode representar um risco de segurança.

### 4.2. O handshake em SSL bidirecional
Com SSL unilateral, o servidor deve confiar em todos os clientes. Porém, o SSL bidirecional adiciona a capacidade do servidor de estabelecer clientes confiáveis ​​também. Durante um handshake bidirecional, o cliente e o servidor devem apresentar e aceitar os certificados públicos um do outro antes que uma conexão bem-sucedida possa ser estabelecida.

# 5. Cenários de falha de handshake
Depois de fazer essa revisão rápida, podemos examinar os cenários de falha com maior clareza.

Um handshake SSL, em comunicação unilateral ou bidirecional, pode falhar por vários motivos. Iremos examinar cada um desses motivos, simular a falha e entender como podemos evitar tais cenários.

Em cada um desses cenários, usaremos o SimpleClient e o SimpleServer que criamos anteriormente.

### 5.1. Certificado de servidor ausente
Vamos tentar rodar o SimpleServer e conectá-lo através do SimpleClient. Enquanto esperamos ver a mensagem “Hello World!”, Nos é apresentada uma exceção:

```
Exception in thread "main" javax.net.ssl.SSLHandshakeException: 
  Received fatal alert: handshake_failure
```

Agora, isso indica que algo deu errado. A SSLHandshakeException acima, de forma abstrata, está informando que o cliente ao se conectar ao servidor não recebeu nenhum certificado.

Para resolver esse problema, usaremos o armazenamento de chaves que geramos anteriormente, passando-os como propriedades do sistema para o servidor:

```
-Djavax.net.ssl.keyStore=clientkeystore.jks -Djavax.net.ssl.keyStorePassword=password
```

É importante observar que a propriedade do sistema para o caminho do arquivo de armazenamento de chave deve ser um caminho absoluto ou o arquivo de armazenamento de chave deve ser colocado no mesmo diretório de onde o comando Java é chamado para iniciar o servidor. A propriedade do sistema Java para armazenamento de chaves não oferece suporte a caminhos relativos.

Isso nos ajuda a obter o resultado que esperamos? Vamos descobrir na próxima subseção.

### 5.2. Certificado de servidor não confiável
Conforme executamos o SimpleServer e o SimpleClient novamente com as alterações na subseção anterior, o que obtemos como saída:

```
Exception in thread "main" javax.net.ssl.SSLHandshakeException: 
  sun.security.validator.ValidatorException: 
  PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: 
  unable to find valid certification path to requested target
```

Bem, não funcionou exatamente como esperávamos, mas parece que falhou por um motivo diferente.

Essa falha específica é causada pelo fato de que nosso servidor está usando um certificado autoassinado que não é assinado por uma Autoridade de Certificação (CA).

Na verdade, sempre que o certificado for assinado por algo diferente do que está no armazenamento confiável padrão, veremos esse erro. O truststore padrão no JDK normalmente é fornecido com informações sobre CAs comuns em uso.

Para resolver esse problema aqui, teremos que forçar o SimpleClient a confiar no certificado apresentado pelo SimpleServer. Vamos usar o armazenamento confiável que geramos anteriormente, passando-os como propriedades do sistema para o cliente:

```
-Djavax.net.ssl.trustStore=clienttruststore.jks -Djavax.net.ssl.trustStorePassword=password
```

Observe que esta não é a solução ideal. Em um cenário ideal, não devemos usar um certificado autoassinado, mas um certificado que foi certificado por uma Autoridade de Certificação (CA) na qual os clientes podem confiar por padrão.

Vamos para a próxima subseção para descobrir se obtemos nossa saída esperada agora.

### 5.3. Certificado de cliente ausente
Vamos tentar mais uma vez executar o SimpleServer e o SimpleClient, tendo aplicado as alterações das subseções anteriores:

```
Exception in thread "main" java.net.SocketException: 
  Software caused connection abort: recv failed
```

Novamente, não é algo que esperávamos. O SocketException aqui nos diz que o servidor não pode confiar no cliente. Isso ocorre porque configuramos um SSL bidirecional. Em nosso SimpleServer temos:

```
((SSLServerSocket) listener).setNeedClientAuth(true);
```

O código acima indica que um SSLServerSocket é necessário para autenticação do cliente por meio de seu certificado público.

Podemos criar um keystore para o cliente e um truststore correspondente para o servidor de maneira semelhante àquela que usamos ao criar o keystore e o truststore anteriores.

Reiniciaremos o servidor e passaremos as seguintes propriedades do sistema:

```
-Djavax.net.ssl.keyStore=serverkeystore.jks \
    -Djavax.net.ssl.keyStorePassword=password \
    -Djavax.net.ssl.trustStore=servertruststore.jks \
    -Djavax.net.ssl.trustStorePassword=password
```

Em seguida, reiniciaremos o cliente passando estas propriedades do sistema:

```
-Djavax.net.ssl.keyStore=clientkeystore.jks \
    -Djavax.net.ssl.keyStorePassword=password \
    -Djavax.net.ssl.trustStore=clienttruststore.jks \
    -Djavax.net.ssl.trustStorePassword=password
```

Finalmente, temos a saída que desejamos:

```
Hello World!
```

### 5.4. Certificados Incorretos
Além dos erros acima, um handshake pode falhar devido a vários motivos relacionados à forma como criamos os certificados. Um erro comum está relacionado a um CN incorreto. Vamos explorar os detalhes do keystore do servidor que criamos anteriormente:

```
keytool -v -list -keystore serverkeystore.jks
```

Quando executamos o comando acima, podemos ver os detalhes do keystore, especificamente o proprietário:

```
...
Owner: CN=localhost, OU=technology, O=isaccanedo, L=city, ST=state, C=xx
...
```

O CN do proprietário deste certificado é definido como localhost. O CN do proprietário deve corresponder exatamente ao host do servidor. Se houver alguma incompatibilidade, isso resultará em uma SSLHandshakeException.

Vamos tentar regenerar o certificado do servidor com CN como algo diferente de localhost. Quando usamos o certificado regenerado agora para executar o SimpleServer e SimpleClient, ele falha imediatamente:

```
Exception in thread "main" javax.net.ssl.SSLHandshakeException: 
    java.security.cert.CertificateException: 
    No name matching localhost found
```

O rastreamento de exceção acima indica claramente que o cliente esperava um certificado com o nome como localhost que não encontrou.

Observe que o JSSE não exige a verificação do nome do host por padrão. Habilitamos a verificação do nome do host no SimpleClient por meio do uso explícito de HTTPS:

```
SSLParameters sslParams = new SSLParameters();
sslParams.setEndpointIdentificationAlgorithm("HTTPS");
((SSLSocket) connection).setSSLParameters(sslParams);
```

A verificação do nome do host é uma causa comum de falha e, em geral, deve ser sempre aplicada para melhor segurança. Para obter detalhes sobre a verificação de nome de host e sua importância na segurança com TLS, consulte este artigo.

### 5.5. Versão SSL incompatível

Atualmente, existem vários protocolos criptográficos, incluindo diferentes versões de SSL e TLS em operação.

Conforme mencionado anteriormente, o SSL, em geral, foi substituído pelo TLS por sua força criptográfica. O protocolo criptográfico e a versão são um elemento adicional com o qual um cliente e um servidor devem concordar durante um handshake.

Por exemplo, se o servidor usa um protocolo criptográfico de SSL3 e o cliente usa TLS1.3, eles não podem concordar com um protocolo criptográfico e um SSLHandshakeException será gerado.

Em nosso SimpleClient, vamos alterar o protocolo para algo que não seja compatível com o protocolo definido para o servidor:

```
((SSLSocket) connection).setEnabledProtocols(new String[] { "TLSv1.1" });
```

Quando executarmos nosso cliente novamente, obteremos uma SSLHandshakeException:

```
Exception in thread "main" javax.net.ssl.SSLHandshakeException: 
  No appropriate protocol (protocol is disabled or cipher suites are inappropriate)
```

O rastreamento de exceção em tais casos é abstrato e não nos diz o problema exato. Para resolver esses tipos de problemas, é necessário verificar se o cliente e o servidor estão usando os mesmos protocolos criptográficos ou compatíveis.

### 5.6. Suite Cipher Incompatível
O cliente e o servidor também devem concordar com o conjunto de criptografia que usarão para criptografar as mensagens.

Durante um handshake, o cliente apresentará uma lista de cifras possíveis para usar e o servidor responderá com uma cifra selecionada da lista. O servidor irá gerar uma SSLHandshakeException se não puder selecionar uma cifra adequada.

Em nosso SimpleClient, vamos mudar o pacote de criptografia para algo que não seja compatível com o pacote de criptografia usado por nosso servidor:

```
((SSLSocket) connection).setEnabledCipherSuites(
  new String[] { "TLS_RSA_WITH_AES_128_GCM_SHA256" });
```

Quando reiniciarmos nosso cliente, obteremos uma SSLHandshakeException:

```
Exception in thread "main" javax.net.ssl.SSLHandshakeException: 
  Received fatal alert: handshake_failure
```

Novamente, o rastreamento de exceção é bastante abstrato e não nos diz o problema exato. A solução para esse erro é verificar os conjuntos de criptografia habilitados usados pelo cliente e servidor e garantir que haja pelo menos um conjunto comum disponível.

Normalmente, os clientes e servidores são configurados para usar uma ampla variedade de conjuntos de criptografia, portanto, é menos provável que esse erro aconteça. Se encontrarmos esse erro, normalmente é porque o servidor foi configurado para usar uma cifra muito seletiva. Um servidor pode optar por impor um conjunto seletivo de cifras por motivos de segurança.

# 6. Conclusão
Neste tutorial, aprendemos sobre como configurar SSL usando soquetes Java. Em seguida, discutimos os handshakes SSL com SSL unilateral e bidirecional. Por fim, analisamos uma lista de possíveis motivos pelos quais os handshakes SSL podem falhar e discutimos as soluções.



