# Задача №1
Имеется база со следующими таблицами:
```php
CREATE TABLE `users` (
    `id`         INT(11) NOT NULL AUTO_INCREMENT,
    `name`       VARCHAR(255) DEFAULT NULL,
    `gender`     INT(11) NOT NULL COMMENT '0 - не указан, 1 - мужчина, 2 - женщина.',
    `birth_date` INT(11) NOT NULL COMMENT 'Дата в unixtime.',
    PRIMARY KEY (`id`)
);
CREATE TABLE `phone_numbers` (
    `id`      INT(11) NOT NULL AUTO_INCREMENT,
    `user_id` INT(11) NOT NULL,
    `phone`   VARCHAR(255) DEFAULT NULL,
    PRIMARY KEY (`id`)
);
```
Напишите запрос, возвращающий имя и число указанных телефонных номеров девушек в возрасте от 18 до 22 лет.
Оптимизируйте таблицы и запрос при необходимости.

#### РЕШЕНИЕ:
```php
SELECT `u`.`name`, COUNT(`p`.`phone`) as `count_phone`
FROM `users` as `u`
LEFT JOIN `phone_numbers` as `p`
ON `u`.`id` = `p`.`user_id`
WHERE 
  (`gender` = 2) 
  AND (TIMESTAMPDIFF(YEAR, DATE_FORMAT(FROM_UNIXTIME(`birth_date`),'%Y-%m-%d %H:%i:%s'),NOW()) BETWEEN 18 AND 22)
GROUP BY `u`.`name`
```

#### Оптимизация
1. Заменить тип данных id c int(11) на serial.  Т.к лучше всего подходит для поля PK
2. Для поля  gender – тип ENUM, чтобы гарантировать, что в БД будут только перечисленные значения и кто-то по ошибке не запишет значение 5, что отразится на точности последующей выборки
3. birth_date — заменить на date. С точки зрения производительности не знаю как это отразится, но визуально с базы читается легче чем набор цифр timestamp.
4. Phone  - заменить на char — экономим память
5. Добавить внешние ключи по полю user_id, а также можно добавить индекс по полям gender,  birth_date
```php
CREATE TABLE `user` (
  `id` serial,
  `name` varchar(255) DEFAULT NULL,
  `gender` enum('0','1','2') NOT NULL DEFAULT '0',
  `birth_date` date NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `phone_numbers12` (
  `id` serial,
  `user_id` bigint(20) UNSIGNED NOT NULL,
  `phone` char(15) DEFAULT NULL COMMENT '15 символов в формате MSISDN',
    PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

ALTER TABLE `phone_numbers`
  ADD CONSTRAINT `fk_user_id` FOREIGN KEY (`user_id`) REFERENCES `user` (`id`) 
  ON UPDATE CASCADE 
  ON DELETE NO ACTION;
```
  
  
 # Задача №2
Имеется строка:
https://www.somehost.com/test/index.html?param1=4&param2=3&param3=2&param4=1&param5=3
Напишите функцию, которая:
1. удалит параметры со значением “3”; 
2. отсортирует параметры по значению; 
3. добавит параметр url со значением из переданной ссылки без параметров (в примере: /test/index.html); 
4. сформирует и вернёт валидный URL на корень указанного в ссылке хоста. 
В указанном примере функцией должно быть возвращено:
https://www.somehost.com/?param4=1&param3=2&param1=4&url=%2Ftest%2Findex.html
#### Решение
```php
/**
 * Возвращает подготовленный url
 * @param string $url
 * @return mixed
 */
function prepareUrl($url) {
    if (filter_var($url, FILTER_VALIDATE_URL) === false) {
        return false;
    }

    // Нарезаем url на части
    $urlChunk = parse_url($url);
    $urlChunk['query'] = preg_replace('~&?param\d*=3~', '', $urlChunk['query']);
    preg_match_all('~param\d=(\d)~', $urlChunk['query'], $paramsArray);

    // Приводим значения параметров к integer и собираем в массиве, для последующей сортировки
    $parametersValue = [];
    foreach ($paramsArray[1] as $valueParam) {
        $parametersValue[] = (int)$valueParam;
    }
    $sortedParameters = array_combine($parametersValue, $paramsArray[0]);
    ksort($sortedParameters);

    return $urlChunk['scheme'] . '://'
        . $urlChunk['host'] . '?'
        . implode('&', $sortedParameters)
        . '&url=' . urlencode ($urlChunk['path']);
}
$url = 'https://www.somehost.com/test/index.html?param1=4&param2=3&param3=2&param4=1&param5=3';
prepareUrl($url);
```


# Задача №3
Напишите код в парадигме ООП, соответствующий следующей структуре.
Сущности: Пользователь, Статья.
Связи: Один пользователь может написать несколько статей. У каждой статьи может быть только один пользователь-автор.
Функциональность:
возможность для пользователя создать новую статью; 
возможность получить автора статьи; 
возможность получить все статьи конкретного пользователя; 
возможность сменить автора статьи. 
Если вы применили какие-либо паттерны при написании, укажите какие и с какой целью.
Код, реализующий конкретную функциональность, не требуется, только общая структура классов и методов.
Код должен быть прокомментирован в стиле PHPDoc.

#### Решение
При написании данного кода применяется шаблон проектирования ActiveRecord, где где объект класса соответствует записи в таблице БД, и который сам себя может сохранять, редактировать.
Подобного рода задачи были рреализованы мной при написании учебного framrwork, ниже ссылка
<https://github.com/scriptius/php2/tree/master/App/Models>

```php
/**
 * Класс для работы с БД
 * Class Model
 */
class Model
{
    /**
     * Соханение модели
     */
    public function save(){}

    /**
     * Удаление модели
     */
    public function delete(){}
}


/**
 * Класс сущности Пользователь
 * Class User
 */
class User extends Model
{
    /**
     * Id пользователя
     * @var int
     */
    public $id;

    /**
     * Имя пользователя
     * @var string
     */
    public $name;

    /**
     * Создать статью
     * @return object Article
     */
    public function createArticle() {
        $article = new Article();
        $article->authorId = $this->id;
        return $article;
    }
}


/**
 * Класс сущности Статья
 * Class Article
 */
class Article extends Model
{
    /**
     * Id автора новости
     * @var int
     */
    public $author_id;

    /**
     * Id новости
     * @var int
     */
    public $id;

    /**
     * Название новости
     * @var int
     */
    public $title;

    /**
     * Получить данные в связанной таблице. Например автора статьи
     * @param $k string Имя недоступного свойства
     * @return mixed Возвлащает или свойство или объект Author
     * (в случае запроса свойста "authors" ) или FALSE - если свойство отсутствует
     */
    public function __get($k)
    {
        if ('author' == $k and !empty($this->author_id)) {
            $author = User::findById((int)$this->author_id);
            return $author;
        } else {
            if (array_key_exists($k, $this->data)) {
                return $this->data[$k];
            } else {
                return false;
            }
        }
    }


    /**
     * Записать данные в связанную таблице. Например сменить автора статьи
     * @param $k string Имя недоступного свойства
     * @return mixed Возвлащает или свойство или объект Author
     * (в случае запроса свойста "authors" ) или FALSE - если свойство отсутствует
     */
    public function __set($k){}


    /**
     * Поиск всех статей автора
     * @param $id integer Id автора
     * @return  object User
     */
    public static function findByAuthor($id){
        return User::findById($id);
    }

}

```


# Задача №4
Проведите рефакторинг, исправьте баги и продокументируйте в стиле PHPDoc код, приведённый ниже (таблица users здесь аналогична таблице users из задачи №1).
Примечание: код написан исключительно в тестовых целях, это не "жизненный пример" :)
```php
function load_users_data($user_ids) {
    $user_ids = explode(',', $user_ids);
    foreach ($user_ids as $user_id) {
        $db = mysqli_connect("localhost", "root", "123123", "database");
        $sql = mysqli_query($db, "SELECT * FROM users WHERE id=$user_id");
        while($obj = $sql->fetch_object()){
            $data[$user_id] = $obj->name;
        }
        mysqli_close($db);
    }
    return $data;
}
// Как правило, в $_GET['user_ids'] должна приходить строка
// с номерами пользователей через запятую, например: 1,2,17,48
$data = load_users_data($_GET['user_ids']);
foreach ($data as $user_id=>$name) {
    echo "<a href=\"/show_user.php?id=$user_id\">$name</a>";
}
```
Плюсом будет, если укажете, какие именно уязвимости присутствуют в исходном варианте (если таковые, на ваш взгляд, имеются), и приведёте примеры их проявления.
#### Решение
1. Данные $user_ids перед вставкой не проверены, одно из первых правил безопасности
2. Я бы переписал данный код с процедурного подхода на объектный
3. Объект класса DB необходимо вынести в отдельный класс и сделать его singleton
4. Нет подготовленных запросов, что может приветсти к SQL-инъекциям
5. Рекомендуется инициализировать первоначальные значения переменных, например $data в load_users_data()
6. Пароль к БД хранится не в конфиге, причем в открытом виде. Я бы сделал конфиг, в котором сделал бы секцию db и при деплое при помощи phing подменял {{USERNAME}} и {{PASSWORD}} на конкретные значения
```php
 'db' => [
    'host'     => 'localhost',
    'dbName'   => 'test',
    'userName' => '{{USERNAME}}',
    'password' => '{{PASSWORD}}',
],
```
Листинг
```php
/**
 * Получение массива UsersIDS
 * @param string $listUsersIDS
 * @return array
 */
function getListUsersIDS (string $listUsersIDS) {
    $userIds = explode(',', $listUsersIDS);
    return checkID($userIds);
}

/**
 * Проверка id на корректность
 * @param array $ids
 * @return array
 */
function checkID (array $ids) {
    $cleanIDS = [];
    foreach ($ids as $itemID) {
        if (is_numeric((int)$itemID)) {
            $cleanIDS[] = $itemID;
        } else {
            continue;
        }
    }
    return $cleanIDS;
}

/**
 * Загрузка пользовательских данных
 * @param $userIds
 * @return array
 */
function load_users_data($userIds) {
    $data = [];
    $config = Config::getInstance();
    // Условно в $dbh хранится объект PDO, в который мы передаем условный конфиг
    $dbh = Db::getInstance($config);
    $sth = $dbh->prepare('SELECT * FROM users WHERE id=:userID');
    foreach (getListUsersIDS($userIds) as $userId) {
        $sth->bindParam(':userID', $userId, PDO::PARAM_INT);
        $sth->execute();
        while($obj = $sth->fetch(PDO::FETCH_OBJ)) {
            $data[$userId] = $obj->name;
        }

        $dbh = null;
    }
    return $data;
}
// Как правило, в $_GET['user_ids'] должна приходить строка
// с номерами пользователей через запятую, например: 1,2,17,48
$data = load_users_data($_GET['user_ids']);
foreach ($data as $user_id => $name) {
    echo "<a href=\"/show_user.php?id=$user_id\">$name</a>";
}
```

# Вопрос №1
Занимались ли вы разработкой автоматизированных функциональных тестов, нагрузочных тестов, либо unit-тестов? Если да – сообщите, что доводилось использовать, какие решали задачи и с каким результатом. Если нет – не страшно, поможем освоить :)

В учебных целях писал тесты на php unit

# Вопрос №2
Перечислите книги (либо веб-ресурсы), которые Вы прочитали и усвоили (либо регулярно читаете), и какие можете рекомендовать своим коллегам для использования в работе, повышения профессионализма, расширения кругозора.
1. Habrahabr
2. Стив Макконел "Совершенный код" - прочитано 75%
3. Мэт Зандстра "PHP объекты, шаблоны и методики программирования"  - прочитано 50%
4. Рекомендую также Эрик Фримен Элизабет Фримен - Паттерны проектирования.
5. Хочу прочитать Мартин фаулер "Рефакторинг"

