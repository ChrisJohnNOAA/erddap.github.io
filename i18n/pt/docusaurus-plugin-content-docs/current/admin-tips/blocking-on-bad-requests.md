# Bloqueio ERDDAP™ solicitações com base no conteúdo do pedido, não com base no IP

Este conteúdo é baseado em um [mensagem de Roy Mendelssohn para o ERDDAP™ grupo de usuários](https://groups.google.com/g/erddap/c/XcvPkoGtchg) .

## O problema

Se você é como nós, você está vendo um monte de bots fazendo pedidos de dados para o seu ERDDAP™ , e as solicitações muitas vezes parecem que são feitas por scripts mal codificados, seja por LLMs (O meu palpite) ou por humanos. Uma coisa sobre LLMs é que eles vão fazer exatamente o que você os pede para fazer, por isso, se você não dizer a eles para ter os códigos de retorno de verificação de script, e não dizer-lhes o que fazer em casos diferentes, então geralmente o código não fará nenhuma dessas coisas. E se lhe disseres para continuar a tentar, será. É o que estamos a ver pelo menos.

Mas aqueles que estão fazendo um número em nosso ERDDAP™ são assim:

 https://coastwatch.pfeg.noaa.gov/erddap/jplMURSST41.parquetWMeta
 

Estes envios ERDDAP™ em terra sem nunca, devido ao melhor que eu posso determinar, que esta URL não está apenas solicitando todo o conjunto de dados, o que é que eu não sei quantos terabytes, mas quer que ele convertido em um arquivo de parquet. Pior as solicitações estão vindo de IPs em movimento, então bloquear IPs é pior do que whack-a-mole, você nunca vai obtê-los todos bloqueados.

Então a pergunta é como você pode bloquear com base no conteúdo do pedido? Antes de ir mais longe, se você é experiência trava pode ser de diferentes causas, então o que estamos vendo - eu passei muito tempo passando por logs e também eu sou notificado quando há uso perigoso de memória alta, e observou quando um acidente seguiu essa notificação, e também notou o que era o denominador comum nesses acidentes, que pode não ser o caso para o seu servidor. Então pesquisa isso antes de tomar qualquer ação, mas isso pode dar-lhe alguma ideia como bloquear esses pedidos.

Só para que eu esteja claro, você pode pensar em um ERDDAP™ pedido como:

 https://baseURL/erddap/method/datasetID.filetype?constraint
 

No nosso caso, o denominador comum dos acidentes foi o griddap ou pedidos de mesa com certos tipos de arquivos, mas sem restrições. Então a pergunta é como bloquear pedidos sem restrições? Exceto não é tão simples, porque há qualquer número de tipos de arquivos que não precisam de um constrangimento e será bem comportado, então não queremos bloquear esses. Depois de falar com Chris, e sem dúvida eu deixei para fora algo, os tipos de arquivos que são bem comportados sem uma restrição são:

- Não.
- INSTITUIÇÕES
- .iso19139_2007
- .iso19115_3_2016
-  .nc CFHeader
-  .nc CFMAHeader
- Não.
- .dds
- .html
- .
- .subset
-  .nc Cabeçalho
- Ajuda
- Não.
- INSTITUIÇÕES
-  .nc O que é? (este será no próximo novo lançamento) .

## Solução

A solução "Eu encontrei" funciona no Apache2, eu não sei nginx, mas eu imagino que haja algo semelhante. No início eu tentei mod_security, mas isso ficou muito complicado e não funcionou muito bem. A solução é usar mod_rewrite. Agora eu sou qualquer coisa, mas um especialista nisso, então enquanto eu defini o que eu estava tentando realizar, a solução, bem como a explicação, é devido a Claude. Ai. O ChatGPT irá fornecer-lhe basicamente a mesma resposta.

Passo 1. Certifique-se de que mod_rewrite está instalado e habilitado. Uma vez que como fazer isso varia com o OS, isso é algo que você pode perguntar ao seu chatbot favorito.

Passo 2. No arquivo apropriado que configura ssl para apache2 (que novamente varia por OS) , por exemplo, pode ser algo como /etc/apache2/sites-enabled/sl.conf, adicione o seguinte sob a definição VirtualHost apropriada (nota se você copiar isso há apenas 4 linhas, a terceira linha pode ser embrulhada, desembrulhar-lo) 

```
RewriteEngine On
RewriteCond %{QUERY_STRING} ^$
RewriteCond %{REQUEST_URI} ^/erddap/(griddap|tabledap)/[^/?]+\\.(?!(?:croissant|iso19115_2|iso19139_2007|iso19115_3_2016|ncCFHeader|ncCFMAHeader|das|dds|html|graph|subset|ncHeader|help|fgdc|iso19115|ncoJsonHeader)$)[A-Za-z0-9_]+$
RewriteRule ^ - [R=429,L]
```

Passo 3. Verifique se a configuração é válida: `Teste de configuração do sudo apache2ctl` 

Passo 4. Reinicie apache2: `sudo systemctl reiniciar apache2` 

Passo 5. Verifique seus registros que nada está sendo bloqueado que não deve ser, e que os pedidos apropriados sem um constrangimento retornam um 429 sem nunca bater seu tomcat

## Explicação

Por que isso funciona e o que isso faz - aqui está a explicação de Claude.ai:

Linha 1 — `Reescrever Engine Em` 
Liga o processamento mod_rewrite para este escopo. Sem ele, as diretivas RewriteCond/RewriteRule abaixo são simplesmente ignoradas.

Linha 2 — `RewriteCond %&#123;QUERY_STRING&#125;` 
Uma condição que deve ser verdadeira antes da regra abaixo se aplica. `%` Está tudo depois do ? na URL de solicitação. ^$ é um regex que significa "início de string imediatamente seguido pelo fim da string" — ou seja, uma string vazia. Portanto, esta condição é verdadeira apenas quando não há nenhuma string de consulta — nenhuma expressão de subconfiguração/constrição no pedido.

Linha 3 — `RewriteCond %&#123;REQUEST_URI&#125; ^/erddap/ (Anúncio grátis para sua empresa |  tabledap ) - Sim. (?&#33; (...) $) [A-Za-z0-9_]` 
Uma segunda condição, verificada contra `%` — o caminho de solicitação literal como o cliente o enviou, sempre o caminho completo, independentemente de onde na configuração esta regra vive (deliberadamente escolhido sobre deixar o padrão RewriteRule em si fazer a correspondência, porque padrão-matching dentro de um ` <Location> ` bloco pode se comportar ambiguamente — veja nota abaixo) . Quebrando o regex:

 `^/erddap/` — deve começar com /erddap/
 (Anúncio grátis para sua empresa |  tabledap ) / — seguido por um dos dois ERDDAP™ métodos de acesso

 `[^/]` — datasetID : um ou mais caracteres que não são / ou ?

 `\\.` — um ponto literal

 ` (?&#33; (:croissant | INSTITUIÇÕES | ... | ncoJsonHeader) $) ` — uma testa negativa: "desde que o que se segue não seja um desses arquivos exatos Digite nomes até o final da cadeia." Estes são o arquivoTypes ERDDAP™ pode legitimamente servir sem restrições (metadados, estrutura, páginas de formulário, etc.) — a testa é o que os exclui de serem bloqueados.

 `[A-Za-z0-9_]` — o arquivo real Extensão de tipo (letras, dígitos, underscore) , obrigado a correr para o fim da cadeia.
Então esta condição é verdadeira somente quando o caminho é um griddap/ tabledap pedido de algum arquivo Digite que não está na lista segura sem restrições.

Linha 4 — `RewriteRule ^ - [R=429,L]` 
A própria regra. Porque ambas as condições acima já devem ser verdadeiras para Apache até mesmo avaliar esta linha, o padrão aqui não precisa verificar qualquer outra coisa — ^ apenas combina "início da cadeia", que é sempre verdade. - significa "não reescrever a URL para nada diferente" (não estamos redirecionando para qualquer lugar, apenas curto-circuito o pedido) . As bandeiras:

 `R=429` — responder com uma ação de redirecionamento HTTP carregando código de status 429 ("Muitos pedidos") em vez de servir o pedido.
L — "Última": pare de processar quaisquer outras regras de reescrever uma vez que este fogo.
Junte-se: se a cadeia de consulta estiver vazia, E o pedido é para um griddap/ tabledap arquivo Tipo que não está na lista segura, imediatamente retornar 429 — sem nunca entrar em contato com o ERDDAP /Tomcat backend.

Porquê? `%` em vez de deixar o padrão RewriteRule combinar o caminho diretamente (vale a pena incluir como uma nota para colegas, uma vez que é a parte não-obvious) : dentro de um ` <Location> ` block, o que um padrão RewriteRule nu realmente é combinado com pode se comportar inconsistentemente dependendo da versão e contexto do Apache. Explicativamente puxando o caminho completo via RewriteCond `%` sidesteps que a ambiguidade inteiramente — é sempre o caminho de solicitação literal, completo, de modo que o regex se comporta exatamente como escrito, independentemente de onde a regra está aninhada.


Eu tenho usado isso há vários dias e parece funcionar muito bem, está bloqueando o que eu estou tentando bloquear e não bloqueando o que eu não quero bloquear. E a nossa ERDDAP™ tornou-se muito mais estável.
