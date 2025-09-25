# WordPress 数据库层实际架构文档

## 概述

本文档详细描述了 WordPress 当前实际使用的数据库层架构。WordPress 采用传统的过程式架构，通过 wpdb 类提供数据库抽象层，使用 MySQLi 扩展与 MySQL/MariaDB 数据库通信。该架构虽然相对简单，但经过 20 年的验证，为全球 43% 的网站提供稳定可靠的服务。

## 核心设计特点

### 1. 架构特征

- **过程式编程**: 以函数为核心的编程模式
- **全局变量**: 大量使用全局变量管理状态
- **直接SQL**: 手工编写原生SQL查询
- **单一数据库**: 仅支持MySQL和MariaDB
- **简单直接**: 避免复杂的抽象层
- **向后兼容**: 保持20年的API稳定性

### 2. 技术栈

```
WordPress应用层 (主题、插件、核心功能)
    ↓ 直接调用
过程式函数 (get_posts, get_user_meta, etc.)
    ↓ 调用
wpdb类 (WordPress数据库抽象层)
    ↓ 使用
MySQLi扩展 (PHP的MySQL接口)
    ↓ 通过TCP/IP
MySQL/MariaDB数据库服务器
```

## WordPress 数据库表结构

### 1. 核心表概览

WordPress 标准安装包含以下核心表：

```php
// WordPress核心表列表 (来自wp-includes/class-wpdb.php)
public $tables = array(
    'posts',           // 文章和页面
    'comments',        // 评论
    'links',          // 友情链接
    'options',        // 网站配置选项
    'postmeta',       // 文章元数据
    'terms',          // 分类术语
    'term_taxonomy',  // 分类法定义
    'term_relationships', // 内容与分类的关系
    'termmeta',       // 分类术语元数据
    'commentmeta',    // 评论元数据
);

// 全局表（用户相关）
public $global_tables = array(
    'users',          // 用户
    'usermeta'        // 用户元数据
);
```

### 2. 详细表结构

#### wp_posts 表 (文章表)
```sql
CREATE TABLE wp_posts (
    ID bigint(20) unsigned NOT NULL auto_increment,
    post_author bigint(20) unsigned NOT NULL default '0',
    post_date datetime NOT NULL default '0000-00-00 00:00:00',
    post_date_gmt datetime NOT NULL default '0000-00-00 00:00:00',
    post_content longtext NOT NULL,
    post_title text NOT NULL,
    post_excerpt text NOT NULL,
    post_status varchar(20) NOT NULL default 'publish',
    comment_status varchar(20) NOT NULL default 'open',
    ping_status varchar(20) NOT NULL default 'open',
    post_password varchar(255) NOT NULL default '',
    post_name varchar(200) NOT NULL default '',
    to_ping text NOT NULL,
    pinged text NOT NULL,
    post_modified datetime NOT NULL default '0000-00-00 00:00:00',
    post_modified_gmt datetime NOT NULL default '0000-00-00 00:00:00',
    post_content_filtered longtext NOT NULL,
    post_parent bigint(20) unsigned NOT NULL default '0',
    guid varchar(255) NOT NULL default '',
    menu_order int(11) NOT NULL default '0',
    post_type varchar(20) NOT NULL default 'post',
    post_mime_type varchar(100) NOT NULL default '',
    comment_count bigint(20) NOT NULL default '0',
    PRIMARY KEY (ID),
    KEY post_name (post_name(191)),
    KEY type_status_date (post_type,post_status,post_date,ID),
    KEY post_parent (post_parent),
    KEY post_author (post_author),
    KEY type_status_author (post_type,post_status,post_author)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**字段说明：**
- `ID`: 文章唯一标识符
- `post_author`: 作者用户ID
- `post_date`: 发布日期（本地时间）
- `post_date_gmt`: 发布日期（GMT时间）
- `post_content`: 文章正文内容
- `post_title`: 文章标题
- `post_status`: 文章状态（publish, draft, private等）
- `post_type`: 内容类型（post, page, attachment等）
- `post_parent`: 父文章ID（用于页面层级）

#### wp_users 表 (用户表)
```sql
CREATE TABLE wp_users (
    ID bigint(20) unsigned NOT NULL auto_increment,
    user_login varchar(60) NOT NULL default '',
    user_pass varchar(255) NOT NULL default '',
    user_nicename varchar(50) NOT NULL default '',
    user_email varchar(100) NOT NULL default '',
    user_url varchar(100) NOT NULL default '',
    user_registered datetime NOT NULL default '0000-00-00 00:00:00',
    user_activation_key varchar(255) NOT NULL default '',
    user_status int(11) NOT NULL default '0',
    display_name varchar(250) NOT NULL default '',
    PRIMARY KEY (ID),
    KEY user_login_key (user_login),
    KEY user_nicename (user_nicename),
    KEY user_email (user_email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### wp_options 表 (配置选项表)
```sql
CREATE TABLE wp_options (
    option_id bigint(20) unsigned NOT NULL auto_increment,
    option_name varchar(191) NOT NULL default '',
    option_value longtext NOT NULL,
    autoload varchar(20) NOT NULL default 'yes',
    PRIMARY KEY (option_id),
    UNIQUE KEY option_name (option_name),
    KEY autoload (autoload)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**字段说明：**
- `option_name`: 配置项名称（如site_url, blogname等）
- `option_value`: 配置项值（可存储序列化数据）
- `autoload`: 是否在页面加载时自动加载（yes/no）

#### 元数据表结构

WordPress使用EAV（Entity-Attribute-Value）模式存储扩展数据：

```sql
-- 文章元数据表
CREATE TABLE wp_postmeta (
    meta_id bigint(20) unsigned NOT NULL auto_increment,
    post_id bigint(20) unsigned NOT NULL default '0',
    meta_key varchar(255) default NULL,
    meta_value longtext,
    PRIMARY KEY (meta_id),
    KEY post_id (post_id),
    KEY meta_key (meta_key(191))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 用户元数据表
CREATE TABLE wp_usermeta (
    umeta_id bigint(20) unsigned NOT NULL auto_increment,
    user_id bigint(20) unsigned NOT NULL default '0',
    meta_key varchar(255) default NULL,
    meta_value longtext,
    PRIMARY KEY (umeta_id),
    KEY user_id (user_id),
    KEY meta_key (meta_key(191))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 评论元数据表
CREATE TABLE wp_commentmeta (
    meta_id bigint(20) unsigned NOT NULL auto_increment,
    comment_id bigint(20) unsigned NOT NULL default '0',
    meta_key varchar(255) default NULL,
    meta_value longtext,
    PRIMARY KEY (meta_id),
    KEY comment_id (comment_id),
    KEY meta_key (meta_key(191))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### 分类系统表

WordPress的分类系统由三个表组成：

```sql
-- 分类术语表
CREATE TABLE wp_terms (
    term_id bigint(20) unsigned NOT NULL auto_increment,
    name varchar(200) NOT NULL default '',
    slug varchar(200) NOT NULL default '',
    term_group bigint(10) NOT NULL default 0,
    PRIMARY KEY (term_id),
    KEY slug (slug(191)),
    KEY name (name(191))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 分类法表
CREATE TABLE wp_term_taxonomy (
    term_taxonomy_id bigint(20) unsigned NOT NULL auto_increment,
    term_id bigint(20) unsigned NOT NULL default 0,
    taxonomy varchar(32) NOT NULL default '',
    description longtext NOT NULL,
    parent bigint(20) unsigned NOT NULL default 0,
    count bigint(20) NOT NULL default 0,
    PRIMARY KEY (term_taxonomy_id),
    UNIQUE KEY term_id_taxonomy (term_id,taxonomy),
    KEY taxonomy (taxonomy)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 对象关系表
CREATE TABLE wp_term_relationships (
    object_id bigint(20) unsigned NOT NULL default 0,
    term_taxonomy_id bigint(20) unsigned NOT NULL default 0,
    term_order int(11) NOT NULL default 0,
    PRIMARY KEY (object_id,term_taxonomy_id),
    KEY term_taxonomy_id (term_taxonomy_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 3. Multisite 表结构

WordPress Multisite 模式增加了以下表：

```sql
-- 站点表
CREATE TABLE wp_blogs (
    blog_id bigint(20) unsigned NOT NULL auto_increment,
    site_id bigint(20) unsigned NOT NULL default '0',
    domain varchar(200) NOT NULL default '',
    path varchar(100) NOT NULL default '',
    registered datetime NOT NULL default '0000-00-00 00:00:00',
    last_updated datetime NOT NULL default '0000-00-00 00:00:00',
    public tinyint(2) NOT NULL default '1',
    archived tinyint(2) NOT NULL default '0',
    mature tinyint(2) NOT NULL default '0',
    spam tinyint(2) NOT NULL default '0',
    deleted tinyint(2) NOT NULL default '0',
    lang_id int(11) NOT NULL default '0',
    PRIMARY KEY (blog_id),
    KEY domain (domain(50),path(5)),
    KEY lang_id (lang_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 网络站点表
CREATE TABLE wp_site (
    id bigint(20) unsigned NOT NULL auto_increment,
    domain varchar(200) NOT NULL default '',
    path varchar(100) NOT NULL default '',
    PRIMARY KEY (id),
    KEY domain (domain(140),path(51))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## wpdb 类架构分析

### 1. wpdb 类的核心结构

```php
// WordPress实际的wpdb类结构 (wp-includes/class-wpdb.php)
class wpdb {
    // 数据库连接信息
    protected $dbuser;
    protected $dbpassword;
    protected $dbname;
    protected $dbhost;
    protected $dbh;         // MySQLi连接句柄

    // 表名属性
    public $posts;
    public $users;
    public $comments;
    public $options;
    public $postmeta;
    public $usermeta;
    // ... 更多表名

    // 查询状态
    public $num_queries = 0;    // 查询计数
    public $last_query;         // 最后执行的查询
    public $last_result;        // 最后的查询结果
    public $last_error = '';    // 最后的错误信息

    // 核心方法
    public function query($query);
    public function get_results($query, $output = OBJECT);
    public function get_row($query, $output = OBJECT, $y = 0);
    public function get_var($query, $x = 0, $y = 0);
    public function get_col($query, $x = 0);

    public function insert($table, $data, $format = null);
    public function update($table, $data, $where, $format = null, $where_format = null);
    public function delete($table, $where, $where_format = null);

    public function prepare($query, ...$args);
    public function _real_escape($data);
}
```

### 2. 数据库连接机制

```php
// WordPress实际的数据库连接过程
public function db_connect($allow_bail = true) {
    $this->is_mysql = true;  // 明确标识只支持MySQL

    // 1. 初始化MySQLi连接
    mysqli_report(MYSQLI_REPORT_OFF);
    $this->dbh = mysqli_init();

    // 2. 解析主机配置
    $host_data = $this->parse_db_host($this->dbhost);
    list($host, $port, $socket, $is_ipv6) = $host_data;

    // 3. 建立连接
    mysqli_real_connect(
        $this->dbh,
        $host,
        $this->dbuser,
        $this->dbpassword,
        null,           // 不立即选择数据库
        $port,
        $socket,
        $client_flags
    );

    // 4. 设置字符集和选择数据库
    $this->set_charset($this->dbh);
    $this->select($this->dbname, $this->dbh);

    return true;
}
```

### 3. SQL执行流程

```php
// WordPress实际的SQL执行机制
public function query($query) {
    if (!$this->ready) {
        return false;
    }

    // 1. 应用查询过滤器
    $query = apply_filters('query', $query);

    // 2. 清空之前的结果
    $this->flush();

    // 3. 安全检查
    if ($this->check_current_query && !$this->check_ascii($query)) {
        $this->last_error = 'WordPress database error: Could not perform query because it contains invalid data.';
        return false;
    }

    // 4. 记录查询
    $this->last_query = $query;

    // 5. 执行SQL
    $this->_do_query($query);

    // 6. 错误处理和重连逻辑
    $mysql_errno = mysqli_errno($this->dbh);
    if (empty($this->dbh) || 2006 === $mysql_errno) {
        if ($this->check_connection()) {
            $this->_do_query($query);
        }
    }

    // 7. 处理结果
    if ($this->last_error) {
        $this->print_error();
        return false;
    }

    return $this->result;
}

// 核心SQL执行方法
private function _do_query($query) {
    // 性能监控
    if (defined('SAVEQUERIES') && SAVEQUERIES) {
        $this->timer_start();
    }

    // 直接使用MySQLi执行
    if (!empty($this->dbh)) {
        $this->result = mysqli_query($this->dbh, $query);
    }

    // 查询计数
    ++$this->num_queries;

    // 记录日志
    if (defined('SAVEQUERIES') && SAVEQUERIES) {
        $this->log_query($query, $this->timer_stop(), $this->get_caller());
    }
}
```

### 4. 参数化查询实现

```php
// WordPress的prepare方法实际实现
public function prepare($query, ...$args) {
    if (is_null($query)) {
        return;
    }

    // 检查占位符
    if (false === strpos($query, '%')) {
        _doing_it_wrong(
            'wpdb::prepare',
            'The query argument of wpdb::prepare() must have a placeholder.',
            '3.9.0'
        );
    }

    // 处理占位符格式
    $allowed_format = '(?:[1-9][0-9]*[$])?[-+0-9]*(?: |0|\'.)?[-+0-9]*(?:\.[0-9]+)?';

    // 移除现有引号
    $query = str_replace("'%s'", '%s', $query);
    $query = str_replace('"%s"', '%s', $query);

    // 转义未识别的百分号
    $query = preg_replace("/%(?:%|$|(?!($allowed_format)?[sdfFi]))/", '%%\\1', $query);

    // 处理参数转义
    $args_escaped = [];
    foreach ($args as $arg) {
        if (is_string($arg)) {
            $args_escaped[] = $this->_real_escape($arg);
        } else {
            $args_escaped[] = $arg;
        }
    }

    // 使用PHP内置函数格式化
    $query = vsprintf($query, $args_escaped);

    return $this->add_placeholder_escape($query);
}
```

## WordPress 数据库操作模式

### 1. 过程式函数封装

WordPress 提供了大量过程式函数来简化数据库操作：

```php
// 实际的WordPress数据库操作函数

// 获取文章
function get_post($post_id) {
    global $wpdb;
    return $wpdb->get_row($wpdb->prepare(
        "SELECT * FROM {$wpdb->posts} WHERE ID = %d",
        $post_id
    ));
}

// 获取用户
function get_user_by($field, $value) {
    global $wpdb;

    $userdata = $wpdb->get_row($wpdb->prepare(
        "SELECT * FROM {$wpdb->users} WHERE $field = %s",
        $value
    ));

    return $userdata ? new WP_User($userdata) : false;
}

// 获取选项
function get_option($option, $default = false) {
    global $wpdb;

    $value = $wpdb->get_var($wpdb->prepare(
        "SELECT option_value FROM {$wpdb->options} WHERE option_name = %s",
        $option
    ));

    return $value !== null ? maybe_unserialize($value) : $default;
}

// 获取文章meta
function get_post_meta($post_id, $key = '', $single = false) {
    global $wpdb;

    if ($key) {
        $meta_value = $wpdb->get_var($wpdb->prepare(
            "SELECT meta_value FROM {$wpdb->postmeta} WHERE post_id = %d AND meta_key = %s",
            $post_id, $key
        ));

        return $single ? maybe_unserialize($meta_value) : array(maybe_unserialize($meta_value));
    }

    // 获取所有meta
    $meta_list = $wpdb->get_results($wpdb->prepare(
        "SELECT meta_key, meta_value FROM {$wpdb->postmeta} WHERE post_id = %d",
        $post_id
    ), ARRAY_A);

    $meta_array = array();
    foreach ($meta_list as $metarow) {
        $meta_array[$metarow['meta_key']][] = maybe_unserialize($metarow['meta_value']);
    }

    return $meta_array;
}
```

### 2. 复杂查询示例

```php
// WordPress实际的复杂查询实现

// WP_Query类的核心查询构建
class WP_Query {
    public function get_posts() {
        global $wpdb;

        // 构建SELECT语句
        $select = "SELECT $wpdb->posts.*";

        // 构建FROM子句
        $from = "FROM $wpdb->posts";

        // 构建JOIN子句
        $join = '';
        if ($this->is_attachment && $this->get('post_parent')) {
            $join .= " LEFT JOIN $wpdb->posts AS p2 ON ($wpdb->posts.post_parent = p2.ID) ";
        }

        // 构建WHERE子句
        $where = "WHERE 1=1";

        // 文章状态
        if (!empty($this->query_vars['post_status'])) {
            $statusin = implode("','", array_map('esc_sql', (array) $this->query_vars['post_status']));
            $where .= " AND $wpdb->posts.post_status IN ('$statusin')";
        }

        // 文章类型
        if (!empty($this->query_vars['post_type'])) {
            $typein = implode("','", array_map('esc_sql', (array) $this->query_vars['post_type']));
            $where .= " AND $wpdb->posts.post_type IN ('$typein')";
        }

        // 作者查询
        if (!empty($this->query_vars['author'])) {
            $where .= " AND $wpdb->posts.post_author = " . intval($this->query_vars['author']);
        }

        // 分类查询
        if (!empty($this->query_vars['cat'])) {
            $join .= " LEFT JOIN $wpdb->term_relationships ON ($wpdb->posts.ID = $wpdb->term_relationships.object_id)";
            $join .= " LEFT JOIN $wpdb->term_taxonomy ON ($wpdb->term_relationships.term_taxonomy_id = $wpdb->term_taxonomy.term_taxonomy_id)";
            $where .= " AND $wpdb->term_taxonomy.term_id IN (" . implode(',', array_map('intval', (array) $this->query_vars['cat'])) . ")";
        }

        // 构建ORDER BY子句
        $orderby = '';
        if (!empty($this->query_vars['orderby'])) {
            switch ($this->query_vars['orderby']) {
                case 'date':
                    $orderby = "$wpdb->posts.post_date";
                    break;
                case 'title':
                    $orderby = "$wpdb->posts.post_title";
                    break;
                case 'menu_order':
                    $orderby = "$wpdb->posts.menu_order";
                    break;
            }
        }

        // 构建LIMIT子句
        $limits = '';
        if (!empty($this->query_vars['posts_per_page']) && $this->query_vars['posts_per_page'] != -1) {
            $pgstrt = absint(($this->query_vars['paged'] - 1) * $this->query_vars['posts_per_page']);
            $limits = "LIMIT $pgstrt, " . intval($this->query_vars['posts_per_page']);
        }

        // 组装最终查询
        $pieces = array('select', 'from', 'join', 'where', 'orderby', 'limits');
        $clauses = compact($pieces);

        // 应用过滤器
        $clauses = apply_filters('posts_clauses', $clauses, $this);

        $sql = "{$clauses['select']} {$clauses['from']} {$clauses['join']} {$clauses['where']} {$clauses['orderby']} {$clauses['limits']}";

        return $wpdb->get_results($sql);
    }
}
```

## 性能特点和限制

### 1. 性能优势

```php
// WordPress数据库层的性能优势

/**
 * 1. 直接SQL执行 - 无ORM开销
 */
$posts = $wpdb->get_results("
    SELECT p.*, u.display_name
    FROM {$wpdb->posts} p
    LEFT JOIN {$wpdb->users} u ON p.post_author = u.ID
    WHERE p.post_status = 'publish'
    ORDER BY p.post_date DESC
    LIMIT 10
");
// 直接执行，性能最优

/**
 * 2. 简单对象结构 - 内存占用低
 */
class wpdb {
    // 只有必要的属性，约2MB内存占用
    protected $dbh;         // MySQLi连接句柄
    public $last_query;     // 最后查询
    public $num_queries;    // 查询计数
    // ... 最小化的属性集
}

/**
 * 3. 针对MySQL优化
 */
// 使用MySQL特定功能
$wpdb->query("SELECT SQL_CALC_FOUND_ROWS * FROM {$wpdb->posts} LIMIT 10");
$total = $wpdb->get_var("SELECT FOUND_ROWS()");

// MySQL特有的全文搜索
$wpdb->get_results($wpdb->prepare("
    SELECT *, MATCH(post_title, post_content) AGAINST(%s IN NATURAL LANGUAGE MODE) as relevance
    FROM {$wpdb->posts}
    WHERE MATCH(post_title, post_content) AGAINST(%s IN NATURAL LANGUAGE MODE)
    ORDER BY relevance DESC
", $search_term, $search_term));
```

### 2. 架构限制

```php
// WordPress数据库层的限制

/**
 * 1. 手写SQL - 开发效率低
 */
// 需要手工构建复杂查询
$sql = "
    SELECT p.*,
           u.display_name,
           COUNT(c.comment_ID) as comment_count
    FROM {$wpdb->posts} p
    LEFT JOIN {$wpdb->users} u ON p.post_author = u.ID
    LEFT JOIN {$wpdb->comments} c ON p.ID = c.comment_post_ID AND c.comment_approved = '1'
    WHERE p.post_status = 'publish'
      AND p.post_type = 'post'
      AND p.post_date >= %s
    GROUP BY p.ID
    ORDER BY p.post_date DESC
    LIMIT %d
";
$posts = $wpdb->get_results($wpdb->prepare($sql, $date, $limit));

/**
 * 2. 无关系管理 - 需要手动JOIN
 */
// 获取文章及其作者信息需要手动JOIN
function get_posts_with_authors() {
    global $wpdb;
    return $wpdb->get_results("
        SELECT p.*, u.display_name as author_name, u.user_email as author_email
        FROM {$wpdb->posts} p
        LEFT JOIN {$wpdb->users} u ON p.post_author = u.ID
        WHERE p.post_status = 'publish'
    ");
}

/**
 * 3. 无类型安全 - 运行时错误
 */
// 可能的SQL注入风险（如果不使用prepare）
$user_id = $_GET['user_id']; // 危险！
$wpdb->get_results("SELECT * FROM {$wpdb->posts} WHERE post_author = $user_id");

// 正确做法
$user_id = intval($_GET['user_id']);
$wpdb->get_results($wpdb->prepare(
    "SELECT * FROM {$wpdb->posts} WHERE post_author = %d",
    $user_id
));

/**
 * 4. 重复代码 - 相似查询需要重写
 */
function get_published_posts() {
    global $wpdb;
    return $wpdb->get_results("SELECT * FROM {$wpdb->posts} WHERE post_status = 'publish' AND post_type = 'post'");
}

function get_published_pages() {
    global $wpdb;
    return $wpdb->get_results("SELECT * FROM {$wpdb->posts} WHERE post_status = 'publish' AND post_type = 'page'");
}
// 大量重复的相似代码
```

## 实际使用示例

### 1. 基础CRUD操作

```php
// WordPress实际的数据库操作示例

/**
 * 创建文章
 */
function create_post_example() {
    global $wpdb;

    $post_data = array(
        'post_title'   => '示例文章',
        'post_content' => '这是文章内容',
        'post_status'  => 'publish',
        'post_type'    => 'post',
        'post_author'  => 1,
        'post_date'    => current_time('mysql')
    );

    $result = $wpdb->insert($wpdb->posts, $post_data);

    if ($result !== false) {
        $post_id = $wpdb->insert_id;

        // 添加文章meta
        $wpdb->insert($wpdb->postmeta, array(
            'post_id'    => $post_id,
            'meta_key'   => '_featured',
            'meta_value' => 'yes'
        ));

        return $post_id;
    }

    return false;
}

/**
 * 读取文章
 */
function get_post_example($post_id) {
    global $wpdb;

    // 获取文章基本信息
    $post = $wpdb->get_row($wpdb->prepare(
        "SELECT * FROM {$wpdb->posts} WHERE ID = %d",
        $post_id
    ));

    if ($post) {
        // 获取文章meta
        $meta = $wpdb->get_results($wpdb->prepare(
            "SELECT meta_key, meta_value FROM {$wpdb->postmeta} WHERE post_id = %d",
            $post_id
        ));

        $post->meta = array();
        foreach ($meta as $m) {
            $post->meta[$m->meta_key] = maybe_unserialize($m->meta_value);
        }

        // 获取作者信息
        $author = $wpdb->get_row($wpdb->prepare(
            "SELECT display_name, user_email FROM {$wpdb->users} WHERE ID = %d",
            $post->post_author
        ));

        $post->author = $author;
    }

    return $post;
}

/**
 * 更新文章
 */
function update_post_example($post_id, $data) {
    global $wpdb;

    $result = $wpdb->update(
        $wpdb->posts,
        $data,
        array('ID' => $post_id),
        null,
        array('%d')
    );

    return $result !== false;
}

/**
 * 删除文章
 */
function delete_post_example($post_id) {
    global $wpdb;

    // 删除文章meta
    $wpdb->delete($wpdb->postmeta, array('post_id' => $post_id), array('%d'));

    // 删除评论
    $wpdb->delete($wpdb->comments, array('comment_post_ID' => $post_id), array('%d'));

    // 删除文章
    $result = $wpdb->delete($wpdb->posts, array('ID' => $post_id), array('%d'));

    return $result !== false;
}
```

### 2. 复杂查询示例

```php
/**
 * 获取热门文章（根据评论数）
 */
function get_popular_posts($limit = 10) {
    global $wpdb;

    return $wpdb->get_results($wpdb->prepare("
        SELECT p.*, u.display_name as author_name,
               COUNT(c.comment_ID) as comment_count
        FROM {$wpdb->posts} p
        LEFT JOIN {$wpdb->users} u ON p.post_author = u.ID
        LEFT JOIN {$wpdb->comments} c ON p.ID = c.comment_post_ID
                                      AND c.comment_approved = '1'
        WHERE p.post_status = 'publish'
          AND p.post_type = 'post'
        GROUP BY p.ID
        ORDER BY comment_count DESC, p.post_date DESC
        LIMIT %d
    ", $limit));
}

/**
 * 搜索文章
 */
function search_posts($search_term, $limit = 20) {
    global $wpdb;

    return $wpdb->get_results($wpdb->prepare("
        SELECT p.*, u.display_name as author_name,
               MATCH(p.post_title, p.post_content) AGAINST(%s IN NATURAL LANGUAGE MODE) as relevance
        FROM {$wpdb->posts} p
        LEFT JOIN {$wpdb->users} u ON p.post_author = u.ID
        WHERE p.post_status = 'publish'
          AND p.post_type = 'post'
          AND (
              p.post_title LIKE %s
              OR p.post_content LIKE %s
              OR MATCH(p.post_title, p.post_content) AGAINST(%s IN NATURAL LANGUAGE MODE)
          )
        ORDER BY relevance DESC, p.post_date DESC
        LIMIT %d
    ", $search_term, "%$search_term%", "%$search_term%", $search_term, $limit));
}

/**
 * 获取分类下的文章
 */
function get_posts_by_category($category_id, $limit = 10) {
    global $wpdb;

    return $wpdb->get_results($wpdb->prepare("
        SELECT p.*, u.display_name as author_name
        FROM {$wpdb->posts} p
        LEFT JOIN {$wpdb->users} u ON p.post_author = u.ID
        INNER JOIN {$wpdb->term_relationships} tr ON p.ID = tr.object_id
        INNER JOIN {$wpdb->term_taxonomy} tt ON tr.term_taxonomy_id = tt.term_taxonomy_id
        WHERE p.post_status = 'publish'
          AND p.post_type = 'post'
          AND tt.taxonomy = 'category'
          AND tt.term_id = %d
        ORDER BY p.post_date DESC
        LIMIT %d
    ", $category_id, $limit));
}
```

## 数据库连接和配置

### 1. 配置文件

WordPress 数据库配置在 `wp-config.php` 中定义：

```php
// wp-config.php 实际配置示例

/** 数据库名 */
define('DB_NAME', 'wordpress_db');

/** 数据库用户名 */
define('DB_USER', 'wp_user');

/** 数据库密码 */
define('DB_PASSWORD', 'secure_password');

/** 数据库主机 */
define('DB_HOST', 'localhost');

/** 数据库字符集 */
define('DB_CHARSET', 'utf8mb4');

/** 数据库校对规则 */
define('DB_COLLATE', '');

/** WordPress表前缀 */
$table_prefix = 'wp_';

/** 开启数据库查询日志（调试用） */
define('SAVEQUERIES', true);

/** 开启数据库错误显示 */
define('WP_DEBUG', true);
define('WP_DEBUG_DISPLAY', true);
```

### 2. 连接池和性能

```php
// WordPress的连接管理特点

/**
 * 单连接模式 - 每个请求一个连接
 */
class wpdb {
    protected $dbh; // 单个MySQLi连接句柄

    public function __construct($dbuser, $dbpassword, $dbname, $dbhost) {
        // 每个WordPress请求创建一个数据库连接
        $this->db_connect();
    }

    // 请求结束时自动关闭连接
    public function __destruct() {
        if ($this->dbh) {
            mysqli_close($this->dbh);
        }
    }
}

/**
 * 连接重试机制
 */
public function check_connection($allow_reconnect = true) {
    if (!empty($this->dbh) && mysqli_query($this->dbh, 'DO 1') !== false) {
        return true; // 连接正常
    }

    if ($allow_reconnect) {
        // 尝试重新连接
        $this->db_connect();
        return $this->check_connection(false);
    }

    return false;
}

/**
 * 性能监控
 */
if (defined('SAVEQUERIES') && SAVEQUERIES) {
    // 记录每个查询的详细信息
    $wpdb->queries[] = array(
        $query,                    // SQL查询
        microtime(true) - $start, // 执行时间
        $this->get_caller(),       // 调用堆栈
        microtime(true),           // 开始时间
        array()                    // 自定义数据
    );
}
```

## MySQLi 扩展的作用

### 1. MySQLi 的技术本质

MySQLi（MySQL Improved）是**PHP的C语言扩展**，作为WordPress与MySQL之间的**底层通信桥梁**：

```php
// WordPress中实际使用的MySQLi函数
mysqli_init()           // 初始化连接对象
mysqli_real_connect()   // 建立连接
mysqli_query()         // 执行SQL查询
mysqli_fetch_object()   // 获取结果对象
mysqli_real_escape_string() // 转义字符串防注入
mysqli_error()          // 获取错误信息
mysqli_affected_rows()  // 获取影响行数
mysqli_insert_id()      // 获取插入ID
mysqli_close()          // 关闭连接
```

### 2. MySQLi vs 其他扩展

| 特性 | MySQLi | PDO | 老mysql扩展 |
|------|--------|-----|------------|
| **数据库支持** | 仅MySQL | 多数据库 | 仅MySQL |
| **面向对象** | ✅ 支持 | ✅ 完全OOP | ❌ 只有过程式 |
| **预处理语句** | ✅ 支持 | ✅ 支持 | ❌ 不支持 |
| **WordPress使用** | ✅ 当前使用 | ❌ 不使用 | ❌ 已弃用 |
| **性能** | 🔥 高性能 | 📊 稍慢 | 🔥 高性能 |

## WordPress数据库的实际特点总结

### 优势
1. **极致简单** - 学习成本低，容易理解和维护
2. **高性能** - 直接SQL执行，无ORM开销
3. **久经考验** - 20年验证，服务全球43%网站
4. **资源轻量** - 最小化内存占用和依赖
5. **MySQL优化** - 充分利用MySQL特性和优化

### 限制
1. **手写SQL** - 开发效率相对较低
2. **单一数据库** - 仅支持MySQL/MariaDB
3. **无类型安全** - 缺乏编译时检查
4. **重复代码** - 相似操作需要重复编写
5. **关系管理** - 需要手动处理表关系

## 总结

WordPress 的数据库架构完美体现了**"简单优于复杂"**的设计哲学。虽然不具备现代ORM的便利性，但其稳定性、性能和易用性使其成为世界上最成功的CMS平台之一。

该架构的核心特点包括：

1. **传统过程式** - 基于函数的简单编程模式
2. **wpdb抽象层** - 提供统一的数据库访问接口
3. **MySQLi驱动** - 高性能的C语言扩展作为通信桥梁
4. **直接SQL** - 手工优化的原生查询，性能优异
5. **EAV模式** - 灵活的元数据扩展机制
6. **表前缀** - 支持多WordPress实例共享数据库

这种架构设计为WordPress的广泛采用和长期稳定运行奠定了坚实的基础，证明了有时候**简单而实用的解决方案**比复杂的现代化架构更有价值。
