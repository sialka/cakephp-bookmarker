# CakePHP Application

## 1. Criando o Projeto

```bash
$ composer create-project --prefer-dist cakephp/app:^3.8 bookmarker
```

### 1.1 Testando

```bash
$ cd\bin
$ cake server -p 8765
http://localhost:8765
```

## 2. Criando o banco de dados

Nome do banco: **cake_bookmarks**
Tabelas: **users, bookmarks, tags, bookmarks_tags**

```sql
CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  email VARCHAR(255) NOT NULL,
  password VARCHAR(255) NOT NULL,
  created DATETIME,
  modified DATETIME
);

CREATE TABLE bookmarks (
  id INT AUTO_INCREMENT PRIMARY KEY,
  user_id INT NOT NULL,
  title VARCHAR(50),
  description TEXT,
  url TEXT,
  created DATETIME,
  modified DATETIME,
  FOREIGN KEY user_key (user_id) REFERENCES users(id)
);

CREATE TABLE tags (
  id INT AUTO_INCREMENT PRIMARY KEY,
  title VARCHAR(255),
  created DATETIME,
  modified DATETIME,
  UNIQUE KEY (title)
);

CREATE TABLE bookmarks_tags (
  bookmark_id INT NOT NULL,
  tag_id INT NOT NULL,
  PRIMARY KEY (bookmark_id, tag_id),
  INDEX tag_idx (tag_id, bookmark_id),
  FOREIGN KEY tag_key(tag_id) REFERENCES tags(id),
  FOREIGN KEY bookmark_key(bookmark_id) REFERENCES bookmarks(id)
);
```

### 2.1 Configuranções de acesso ao Database

File: **config/app.php**

```php
return [
  // Mais configuração acima.
  'Datasources' => [
    'default' => [
    'className' => 'Cake\Database\Connection',
    'driver' => 'Cake\Database\Driver\Mysql',
    'persistent' => false,
    'host' => 'localhost',
    'username' => 'root',
    'password' => 'suporte',
    'database' => 'cake_bookmarks',
    'encoding' => 'utf8',
    'timezone' => 'UTC',
    'cacheMetadata' => true,
    ],
  ],
  // Mais configuração abaixo.
];
```

### 2.2 Testando o database

Reinicie o servidor para reconhecer as configurações de acesso ao database.

```bash
$ cake server
```

## 3. Estrutura MVC

Para criar a estrutura mvc usamos o bake console.

```bash
$ cake bake all users
$ cake bake all bookmarks
$ cake bake all tags
```

Para testa, reinicie o servidor e vá para http://localhost:8765/bookmarks

--Continuar Pag. 55--

## 4. Mudando Comportamento no Cadastro

Neste exemplo o nosso usuário está salvando a senha sem criptografia.

Para mudar esse comportamento precisamos fazer no Entity do Model, pois é um comportamento que se dará sempre que a senha for criada ou alterada.

Em **src/Model/Entity/User.php** adicione o seguinte:

```php
namespace App\Model\Entity;

use Cake\ORM\Entity;
use Cake\Auth\DefaultPasswordHasher;

class User extends Entity
{
  // Code from bake.
  protected function _setPassword($value)
  {
    $hasher = new DefaultPasswordHasher();
    return $hasher->hash($value);
  }
}
```

## 5. Criando uma nova rota

Geralmente quando criamos uma rota, estamos dizendo para aplicação que precisamos fazer alguma coisa que invariavelmente tem relação com o database e que nos devolva uma resposta.

Para concluir essa tarefa, precisar completar 04 tarefas:

- 1. Criar a rota.
- 2. Criar o método no controller.
- 3. Implementar o método no table do controller (se houve necessidade).
- 4. Criar o template.

### 5.1 Criando uma Rota

Para criar novas rotas editamos o arquivo **config/routes.php**.

Vamos adicionar o rota **/bookmarks/tagged** veja a url completa:

http://localhost:8765/bookmarks/tagged

```php
Router::scope(
  '/bookmarks',
  ['controller' => 'Bookmarks'],
    function ($routes) {
      $routes->connect('/tagged/*', ['action' => 'tags']);
    }
);
```

Porem ao testar teremos um erro, precisamos criar o controller e a view.

### 5.2 Editando o Controller

Para editar o controller editamos o arquivo **src/Controller/BookmarksController.php**

```php
...
public function tags()
{
  $tags = $this->request->getParam('pass');
  $bookmarks = $this->Bookmarks->find('tagged', [
    'tags' => $tags
  ]);
  $this->set(compact('bookmarks', 'tags'));
}
...
```

### 5.3 Criando Método localizador

Neste exemplo criamos uma rota que recebe parametros, essa url devera listar todas os booksmarks que contem as tags que estão nos parametros.

Entretanto precisamos informar ao controller que ele devera buscar essas informações no database.

Para isso criamos o método **findTagged** no model **BookmarksTable**.
Vamos editar o arquivo **src/Model/Table/BookmarksTable.php**.

```php
...
public function findTagged(Query $query, array $options)
{
  $bookmarks = $this->find()
    ->select(['id', 'url', 'title', 'description']);

  if (empty($options['tags'])) {
    $bookmarks
      ->leftJoinWith('Tags')
      ->where(['Tags.title IS' => null]);
  } else {
    $bookmarks
      ->innerJoinWith('Tags')
      ->where(['Tags.title IN ' => $options['tags']]);
  }

  return $bookmarks->group(['Bookmarks.id']);
}
...
```

### 5.4 Criando a View

Para criar a view criaremos o arquivo **tags.ctp**, que deverá ficar na pasta
**src/Template/Bookmarks/tags.ctp**, seguindo a conveção do CakePHP.

```php
<h1>
  Bookmarks tagged with
  <?= $this->Text->toList(h($tags)) ?>
</h1>

<section>
<?php foreach ($bookmarks as $bookmark): ?>
  <article>
    <h4><?= $this->Html->link($bookmark->title, $bookmark->url) ?></h4>
    <small><?= h($bookmark->url) ?></small>
    <?= $this->Text->autoParagraph(h($bookmark->description)) ?>
  </article>
<?php endforeach; ?>
</section>
```

## 6. Adicionando Login

Para criar o Login faremos atravez de um componente.
Como o acesso ao sistema depende do usuário estar logado, vamos alterar o AppController.

### 6.1 Editando o AppController

**Em src/Controller/AppController.php**

```php
namespace App\Controller;

use Cake\Controller\Controller;

class AppController extends Controller
{
  public function initialize()
  {
    $this->loadComponent('Flash');
    $this->loadComponent('Auth', [
      'authenticate' => [
        'Form' => [
          'fields' => [
            'username' => 'email',
            'password' => 'password'
          ]
        ]
      ],
      'loginAction' => [
        'controller' => 'Users',
        'action' => 'login'
      ]
    ]);
    // Permite a ação display, assim nosso pages controller
    // continua a funcionar.
    $this->Auth->allow(['display']);
    }
  }
```

### 6.2 Editando o UsersController

**Em src/Controller/UsersController.php**

Criando os métodos de login e logou no controller de Users.

```php
<?php

namespace App\Controller;

use App\Controller\AppController;

class UsersController extends AppController
{
  ...
    public function login()
    {
        if ($this->request->is('post')) {
            $user = $this->Auth->identify();
            if ($user) {
                $this->Auth->setUser($user);
                return $this->redirect($this->Auth->redirectUrl());
            }
            $this->Flash->error('Your username or password is incorrect.');
        }
    }

    public function logout()
    {
        $this->Flash->success('You are now logged out.');
        return $this->redirect($this->Auth->logout());
    }
  ...
}
```

### 6.3 Criando o Template de Login

Em **src/Template/Users/login.ctp** adicione o seguinte trecho:

```html
<h1>Login</h1>
<?= $this->Form->create() ?>
<?= $this->Form->input('email') ?>
<?= $this->Form->input('password') ?>
<?= $this->Form->button('Login') ?>
<?= $this->Form->end() ?>
```

### 6.4 Permitindo cadastro de novos usuarios

Para que consigamos criar um usuario para utilizar o sistema vamos informar que a rota /users/add não refer autenticação.

**Em src/Controller/UsersController.php**

```php
public function beforeFilter(\Cake\Event\Event $event)
{
  $this->Auth->allow(['add']);
}
```

## 7. Restringindo Acesso

Em **AppController** adicione o seguinte:

```php
public function isAuthorized($user)
{
  return false;
}
```

Adicionar na function initialize() o **'authorize' => 'Controller'**.

```php
<?php
namespace App\Controller;

...

class AppController extends Controller
{

  public function initialize()
  {

    $this->loadComponent('Flash');
    $this->loadComponent('Auth', [
      'authorize' => 'Controller',
      'authenticate' => [
        'Form' => [
          'fields' => [
              'username' => 'email',
              'password' => 'password'
              ]
          ]
      ],
      'loginAction' => [
        'controller' => 'Users',
        'action' => 'login'
      ]
    ]);
    ...
  }
  ...
}
```

Em seu BookmarksController adicione o seguinte:

```php
<?php

namespace App\Controller;

use App\Controller\AppController;

class BookmarksController extends AppController
{
  ...

  public function isAuthorized($user)
  {
      $action = $this->request->params['action'];
      // As ações add e index são permitidas sempre.
      if (in_array($action, ['index', 'add', 'tags'])) {
          return true;
      }
      // Todas as outras ações requerem um id.
      if (!$this->request->getParam('pass.0')) {
          return false;
      }
      // Checa se o bookmark pertence ao user atual.
      $id = $this->request->getParam('pass.0');
      $bookmark = $this->Bookmarks->get($id);
      if ($bookmark->user_id == $user['id']) {
          return true;
      }
      return parent::isAuthorized($user);
  }
```

pagina 61
