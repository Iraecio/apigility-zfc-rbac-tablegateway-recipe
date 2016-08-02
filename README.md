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

## Instalando

Essa configuração assume que seu Modulo se chama "SeuApp". Então renomei de acordo com o seu.

Instalando o ZfcRbac module.
Faça o download do moculo e copie para dentro /vendor/zf-commons/zfc-rbac/
 
```sh
$ php composer.phar require zf-commons/zfc-rbac:~2.4
```
## Configuração

Adicione o modulo em /config/application.config.php.
E ative o modulo para ser usado pelo ZF2

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

Agora pode remover as configuração 'zf-mvc-auth/authorization' de /module/SeuApp/config/module.config.php - porque nao será mais utilizado.

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
Até agora ja decidimos como vai funcionar nosso sistema, configuramos as funções, tambem definimos cada controller, quem tem direito de realizar cada request, depois alteramos as rotas default do mvc-auth, a authenticação, autorização, e identidade.

Entao vamos continuar...

Agora vá ate o modulo de "SeuApp" e altere o Bootstrap adicionando o evento post de autenticação /module/YourApp/Module.php.

```php
class Module implements ApigilityProviderInterface
{
    public function onBootstrap(MvcEvent $e)
    {
        $eventManager = $e->getApplication()->getEventManager();
        $moduleRouteListener = new ModuleRouteListener();
        $moduleRouteListener->attach($eventManager);
        
        $sm = $e->getApplication()->getServiceManager();
        $eventManager->attach(MvcAuthEvent::EVENT_AUTHENTICATION_POST, $sm->get('SeuApp\\Rbac\\AuthenticationListener'), 100);
    }
    // outras configuração abaixo
}
```

Certifica-se que foi chamado o namespace no topo do arquivo ```use ZF\MvcAuth\MvcAuthEvent; ```

Agora sempre que for preciso fazer uma autenticação sera chamado o servico ``` 'SeuApp\\Rbac\\AuthenticationListener' ```

Então vamos criar ele, Vá até a pasta de "SeuApp\\src\\SeuApp" e crie a pasta ```Rbac```

## AuthenticationListenerFactory

Cria uma classe com o nome ``` AuthenticationListenerFactory.php ``` que será chamado antes de iniciar a class ``` AuthenticationListener.php ```
Está classe e responsável por definir o tableGateway para trabalhar com a tabela ``oauth_users`` ou qualquer outra tabela que esteja usando como deposito de usuarios.

```php

namespace SeuApp\Rbac;

use SeuApp\OAuth\OAuthUserEntity;
use Zend\Db\ResultSet\ResultSet;
use Zend\Db\TableGateway\TableGateway;
use Zend\ServiceManager\ServiceManager;

class AuthenticationListenerFactory
{
    public function __invoke(ServiceManager $services)
    {
        /** @var  $bd \Zend\Db\Adapter\AdapterInterface*/
        $bd = $services->get('db');

        /** @var  $entity  \ArrayObject */
        $entity = new OAuthUserEntity();

        $resultSet = new ResultSet();
        $resultSet->setArrayObjectPrototype($entity);

        $tableGateway = new TableGateway('oauth_users', $bd, null, $resultSet);

        $listener = new AuthenticationListener();
        $listener->setTable($tableGateway);
        
        return $listener;
    }
}
```
## AuthenticationListener

Agora crie a classe com o nome ``` AuthenticationListener.php ``` 

```php
 
namespace SeuApp\Rbac;

use Zend\Db\TableGateway\TableGateway;
use ZF\MvcAuth\Identity\AuthenticatedIdentity;
use ZF\MvcAuth\MvcAuthEvent;

class AuthenticationListener
{
    /** @var  TableGateway */
    public $tableGateway;

    public function setTable(TableGateway $tableGateway)
    {
        $this->tableGateway = $tableGateway;
    }

    public function __invoke(MvcAuthEvent $mvcAuthEvent)
    {
        $identity = $mvcAuthEvent->getIdentity();
        if ($identity instanceof AuthenticatedIdentity) {
            $userId = $identity->getRoleId();
            $oauthUserEntity = $this->tableGateway->select(['username' => $userId])->current();
            $identity->setName($oauthUserEntity->role);
        }
        return $identity;
    }
}
```
Ela e responsavel por receber a identidade do usuario através do ``userId`` entao vamos alterar-lo para receber a função do usuario.

A parte da authenticação é isso, porem ainda não esta funcionando, até porque ainda não criamos a Entity do usuario nem alteramos a tabela mysql dos users.

## IdentityProviderFactory

Esta class e responsável por injetar qual é o modulo de authenticação que será utilizado.

```php
namespace SeuApp\Rbac;

use \Zend\ServiceManager\ServiceManager;

class IdentityProviderFactory
{
    public function __invoke(ServiceManager $services)
    {
        /** @var \Zend\Authentication\AuthenticationService $authenticationProvider */
        $authenticationProvider = $services->get('authentication');

        $identityProvider = new IdentityProvider();
        $identityProvider->setAuthenticationProvider($authenticationProvider);
        
        return $identityProvider;
    }
}
```
Não tem muito segredo, apenas pede o servico authentication, inicia a class identityProvider e seta o authentication para ser ultilizado.

## IdentityProvider

Classe IdentityProvider fornece objeto de identidade exigido pelo RBAC.

O normal do Oauth2 e retornar o userId, mas com Rbac precisamos retorna tambem a função. Por isso vamos personalizar o Provedor de identidade.

```php
namespace SeuApp\Rbac;

use Zend\Authentication\AuthenticationService;
use ZfcRbac\Identity\IdentityProviderInterface;

class IdentityProvider implements IdentityProviderInterface
{
    /** @var Identity $rbacIdentity */
    private $rbacIdentity = null;

    /* @var \Zend\Authentication\AuthenticationService $authenticationProvider */
    private $authenticationProvider;
    
    /**
    * Esta função e chamado pelo IdentityProviderFactory 
    *
    */
    public function setAuthenticationProvider(AuthenticationService $authenticationProvider)
    {
        $this->authenticationProvider = $authenticationProvider;
        return $this;
    }

    /**
     * Checa se o usuarios é authenticado, se for sim, busca a funcao no banco de dados e retorna a identidade.
     *
     * @return Identity
     */
    public function getIdentity()
    {
        if ($this->rbacIdentity === null) {
            $this->rbacIdentity = new Identity();
            $mvcIdentity = $this->authenticationProvider->getIdentity();
            $role = $mvcIdentity->getRoleId();
            $this->rbacIdentity->setRoles($role);
        }
        return $this->rbacIdentity;
    }
}
```

Até aqui tudo bem... mas ainda não esta funcionando. vamos criar a Identity.

## Identity

```php

namespace SeuApp\Rbac;

use ZfcRbac\Identity\IdentityInterface;

class Identity implements IdentityInterface
{
    private $roles = array();

    public function setRoles($roles)
    {
        if (!is_array($roles)) {
            $roles = array($roles);
        }
        $this->roles = $roles;
        
        return $this;
    }

    /**
     * Retorna a lista de funções desta identidade.
     *
     * @return string[]|\Rbac\Role\RoleInterface[]
     */
    public function getRoles()
    {
        return $this->roles;
    }
}
```
 Está classe não tem muito o que se falar e praticamente auto explicativa, define função e retorna elas.
 
 Quase lá vamos parti para a parte da autorização. a parte que tanto queremos, se o usuario tem permisão de ultilizar o que quer recebe a autorização :D
 
 Então vamos criar o Factory dela
 
 ## AuthorizationFactory
 
 Responsavel por injetar o modulo de autorização do Rbac e as configurações feitas la no inicio do zfc_rbac
 
 ```php
 
namespace SeuApp\Rbac;

use Zend\ServiceManager\ServiceManager;

class AuthorizationFactory
{
    public function __invoke(ServiceManager $services)
    {
    
        /** @var \ZfcRbac\Service\AuthorizationService $authorizationService */
        $authorizationService = $services->get('ZfcRbac\Service\AuthorizationService');

        $config = $services->get('config');
        $rbacConfig = $config['zfc_rbac'];
        $authorization = new Authorization();
        $authorization->setConfig($rbacConfig);
        $authorization->setAuthorizationService($authorizationService);

        return $authorization;
    }
}
 ```

## Authorization

```php

namespace SeuApp\Rbac;

use ZF\MvcAuth\Authorization\AuthorizationInterface;
use ZF\MvcAuth\Identity\IdentityInterface;
use ZfcRbac\Service\AuthorizationService;

class Authorization implements AuthorizationInterface
{
    /** @var  AuthorizationService */
    private $authorizationService;
    private $config = [];

    public function setConfig(array $config)
    {
        $this->config = $config; 
    }

    public function setAuthorizationService(AuthorizationService $authorizationService)
    {
        $this->authorizationService = $authorizationService; 
    }

    /**
     * @param IdentityInterface $identity
     * @param mixed $resource
     * @param mixed $privilege
     * @return bool
     */
    public function isAuthorized(IdentityInterface $identity, $resource, $privilege)
    { 
        //aqui busca todas as config do zfc_rbac e seleciona o array 'rest_guard'
        $restGuard = $this->config['rest_guard'];
        
        list($controller, $group) = explode('::', $resource); 
        //verifica se existe a rota no rest_guard se existir ele avalia
        if (isset($restGuard[$controller][$group][$privilege])) { 
            //verifica se o valor e apenas true  ou false ou se tem funçoes definida
            $result = $restGuard[$controller][$group][$privilege];
            //se for array ele vai verificar se sua permissão bate com o que foi definido
            if (is_array($result)) {
                $and = true;
                foreach ($result as $r) { 
                    $and = $and && $this->authorizationService->isGranted($r);  
                }
                $result = $and;
            } 
            return $result;
        }
        
        return true;
    }

}
```

Bem simples neh porem muito eficaz.

agora vamos fexar o capu do carro e ve se funciona para isso vamos as ultimas configuração.

Vá ate o banco de dados e altere a tabela oauth_users ou a que vc utiliza, e adicione o campo ``role VARCHAR(20)``

e entao vamos criar a Entity do usuario para que o TableGateway nao der erro e possa trabalhar com a tabela sem problemas.

## OAuthUserEntity

```php
namespace SeuAPP\OAuth;
 
class OAuthUserEntity
{
    // Arvore das funcoes definidas la em /config/autoload/zfc_rbac.global.php
        // role tree is in /config/autoload/zfc_rbac.global.php
    const ROLE_ADMIN = 'admin';
    const ROLE_USER  = 'user';
    const ROLE_GUEST = 'guest';

    const PERMISSION_CAN_DO_FOO = 'canDoFoo';
    const PERMISSION_CAN_DO_BAR = 'canDoBar';
    const PERMISSION_CAN_DO_BAZ = 'canDoBaz';

    public $user_id;
    public $username;
    public $password;
    public $role;

    public function getArrayCopy()
    {
        return get_object_vars($this);
    }

    public function exchangeArray(Array $data)
    {
        foreach ($data as $key => $value) {
            if (in_array($key, array_keys(get_object_vars($this)))) {
                $this->$key = $value;
            }
        }
    }
     
}
```

E isto e tudo ja temos um modulo de autorização funcionando de acordo com a função de cada um.

## Se quiser força a verificação em cada Resource
Em seu ResourceFactory adicione o serviço de autorização


```php
namespace SeuApp\V1\Rest\Foo;

class FooResourceFactory
{
    /**
     * @param \Zend\ServiceManager\ServiceManager $services
     *
     * @return PluginResource
     */
    public function __invoke($services)
    {
        $db = $services->get('db');
        $entity = new FooEntity();
        $resultSet = new ResultSet();
        $resultSet->setArrayObjectPrototype($entity);
        
        $tableGateway = new TableGateway('fooTable', $db, null, $resultSet);
        
        /** @var \ZfcRbac\Service\AuthorizationService $authorizationService */
        $authorizationService = $services->get('ZfcRbac\Service\AuthorizationService');

        $fooResource = new FooResource();
        $fooResource->setTableGateway($tableGateway);
        $fooResource->setAuthorizationService($authorizationService);

        return $fooResource;
    }
}
```

```php
namespace SeuApp\V1\Rest\Foo;

use ZF\ApiProblem\ApiProblem;
use ZfcRbac\Service\AuthorizationService;

class FooResource extends AbstractResourceListener
{
    /** @var AuthorizationService */
    protected $authorizationService;
    
    /** @var TableGateway */
    protected $tableGateway;
    
    public function setTableGateway(TableGateway $tableGateway)
    {
        $this->tableGateway = $tableGateway;
        return $this;
    }
    
    public function setAuthorizationService(AuthorizationService $authorizationService)
    {
        $this->authorizationService = $authorizationService;
        return $this;
    }

    public function create()
    {
        $authResult = $this->authorizationService->isGranted(OAuthUserEntity::PERMISSION_CAN_DO_FOO);
        if (!$authResult) {
            return new ApiProblem(403, 'Voce não tem permissão para criar Foo.');
        }
        // Voce tem permissão para criar foo
    }
}
```
Isso e tudo.

------------------------------------------------------------------------------------------------------------
Este tutorial foi baseado no original do @remiq

[Veja o tutorial com Doctrine!](remiq/apigility-zfc-rbac-recipe)

Sinta-se livre para aprimorar e contribuir com Rbac Apigility

Se te ajudou da um Stars no projeto gastei um tempo pra criar :D


