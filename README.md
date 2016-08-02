# Apigility and ZfcRbac integration com TableGateway
Você criou aplicativo API com Apigility, com autenticação em OAuth2 integrada e agora você quer adicionar funções? Azar, você não iria encontrar nenhum tutorial de como fazê isso. Até agora.

## Requirementos

- Trabalhando com Apigility API
- TableGateway
- Trabalhando com authenticação Oauth2

## Como isso vai funcionar?

Usaremos Zf-mvc-auth para lidar com a autenticação OAuth2. Vamos injetar nosso listener para postar o evento de autenticação, de modo após a autenticação bem sucedida vamos consultar o Banco de Dados e obter a função do usuário em vez do ID.

In ZfcRbac configuration we point to our IdentityProvider that will translate zf-mvc-auth Identity into 
ZfcRbac Identity.

Vamos adicionar um alias `ZF\MvcAuth\Authorization\AuthorizationInterface` para nosso Authorization, entao o metodo isAuthorized será chamado invés de Acl.

## Installando

Essa configuração assume que seu Modulo se chama "SeuApp". Então renomei de acordo com o seu.

Instalando o ZfcRbac module.
Faça o download do moculo e copie para dentro /vendor/zf-commons/zfc-rbac/
 
```sh
$ php composer.phar require zf-commons/zfc-rbac:~2.4
```

Adicione o modulo em /config/application.config.php.
E ative o modulo para ser usado pelo by ZF2

```php
return array(
    'modules' => array(
        // alguns modulos aqui como. Doctrine
        'ZfcRbac',
        // alguns modulos aqui como. Application, ZF\\Apigility
    )
);
```

Copie /vendor/zf-commons/zfc-rbac/config/zfc_rbac.global.php.dist para /config/autoload/zfc_rbac.global.php.
Este será a base da configuração que usaremos nos proximos passos.

Configure os seguintes valores em /config/autoload/zfc_rbac.global.php.
Isto irá ativar autorização apenas nos locais especificos do codigo.
Se você deseja bloquear todos os controlhes, Leia mais sobre Zfc_rbac guards.

```php
return array(
    'zfc_rbac' => array(
        'identity_provider'   => 'YourApp\\Rbac\\IdentityProvider',
        'guest_role' => 'guest',
        'guards' => array(),
        'protection_policy' => \ZfcRbac\Guard\GuardInterface::POLICY_ALLOW,
    )
);
```

Definir arvore de funçoes em /config/autoload/zfc_rbac.global.php (vamos atualizar o mesmo arquivo)
Isto define que a sua aplicação tem três funções: admin, user e guest.
User tera permissoes "canDoFoo", "canDoBar". 
Admin tera todas as permissoes do user ("canDoFoo", "canDoBar") e suas propria permissao "canDoBaz".
O guest nao recebe nenhuma permissao "assim qualquer request feito sem autenticação e um guest"

```php
return array(
    'zfc_rbac' => array(
        // nossas definições anteriores aqui
        'role_provider' => array(
            'ZfcRbac\Role\InMemoryRoleProvider' =>  array(
                'admin' =>  array(
                    'children'  =>  array('user'),
                    'permissions'   =>  array(
                        'canDoBaz',
                    ),
                ),
                'user' =>  array(
                    'children'  =>  array('guest'),
                    'permissions'   =>  array(
                        'canDoFoo',
                        'canDoBar',
                    ),
                ),
                'guest' =>  array(),
            ),
        ),
    )
);
```

Vamos definir o REST guard em /config/autoload/zfc_rbac.global.php (Atualizando o mesmo arquivo).
Esta parte e parecida com zf-mvc-auth/authorization opção de configuração, ao invés de opçoes booleanas como (true: precisa de autorização, false: aceita qualquer request) usaremos booleanas e arrays de permições (true: aceita qualquer request, false: nao aceita nenhum request, array: aceita so request das permições selecionadas).

```php
    'rest_guard' => [
        'SeuApp\\V1\\Rest\\Foo\\Controller' => [
            'entity' => [
                'GET' => true,              // todo mundo pode usar GET /foo/:id
                'POST' => false,            // ninguem pode usar POST /foo/:id
                'PATCH' => ['canDoFoo'],    // somente admin ou user pode usar PATCH /foo/:id
                'PUT' => ['canDoFoo', 'canDoBar'], // so os usuarios que tem estas permições (admin/user) pode usar PUT /foo/:id 
                'DELETE' => ['canDoFoo'],
            ],
            'collection' => [
                'GET' => true,          // todo mundo pode usar GET /foo
                'POST' => ['canDoFoo'], // so admin ou user pode usar POST /foo 
                'PATCH' => false,       // ninguem pode usar PATCH /foo
                'PUT' => false,
                'DELETE' => ['canDoBaz'], // so admin pode usar DELETE /foo
            ],
        ],
    ],
```

Agora pode remover as configuração 'zf-mvc-auth/authorization' de /module/SeuApp/config/module.config.php - porque nao será mais ultilizado.

em /module/SeuApp/config/module.config.php adicione o seguinte:

```php
return array(
    'service_manager' => array(
        'aliases' => array(
            'ZF\MvcAuth\Authorization\AuthorizationInterface' => 'SeuApp\\Rbac\\Authorization',
        ),
        'factories' => array(
            'SeuApp\\Rbac\\IdentityProvider'   =>  'SeuApp\\Rbac\\IdentityProviderFactory',
            'SeuApp\\Rbac\\AuthenticationListener'  =>  'SeuApp\\Rbac\\AuthenticationListenerFactory',
            'SeuApp\\Rbac\\Authorization'  =>  'SeuApp\\Rbac\\AuthorizationFactory',
        ),
    ),
);
```
Amanhan termino to com sono ja...
