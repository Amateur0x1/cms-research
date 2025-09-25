# Strapi 数据库抽象层详细设计

## 概述

Strapi的数据库抽象层是其架构中的核心组件，提供了统一的数据访问接口，支持多种数据库系统，并实现了现代化的查询构建、关系管理和性能优化功能。本文档深入分析其设计架构和实现细节。

## 整体架构设计

### 1. 分层架构

```
┌─────────────────────────────────────────────┐
│           Application Layer                 │
│     (Controllers, Services, Plugins)       │
├─────────────────────────────────────────────┤
│          Entity Service Layer               │
│    (strapi.entityService.findMany...)      │
├─────────────────────────────────────────────┤
│           Query Engine Layer                │
│      (Query Builder, Relations)            │
├─────────────────────────────────────────────┤
│         Database Adapter Layer              │
│  (PostgreSQL, MySQL, SQLite, MongoDB)      │
├─────────────────────────────────────────────┤
│            Database Driver                  │
│    (Knex.js, Mongoose, Native Drivers)     │
└─────────────────────────────────────────────┘
```

### 2. 核心组件

#### Database Manager
```javascript
// @strapi/database/lib/index.js
class Database {
  constructor(config) {
    this.config = config;
    this.connection = null;
    this.dialect = null;
    this.queryBuilder = null;
    this.entityManager = null;
    this.metadata = new Map();
  }

  async initialize() {
    // 初始化数据库连接
    this.connection = await this.createConnection();
    this.dialect = this.createDialect();
    this.queryBuilder = new QueryBuilder(this);
    this.entityManager = new EntityManager(this);
  }

  query(uid) {
    return this.queryBuilder.query(uid);
  }
}
```

## 数据库适配器系统

### 1. 适配器接口设计

```javascript
// 统一的适配器接口
class DatabaseAdapter {
  constructor(config) {
    this.config = config;
    this.client = null;
  }

  // 连接管理
  async connect() { throw new Error('Must implement connect'); }
  async disconnect() { throw new Error('Must implement disconnect'); }

  // 查询执行
  async executeQuery(sql, params) { throw new Error('Must implement executeQuery'); }

  // 事务支持
  async transaction(callback) { throw new Error('Must implement transaction'); }

  // Schema管理
  async createTable(tableName, definition) { throw new Error('Must implement createTable'); }
  async alterTable(tableName, changes) { throw new Error('Must implement alterTable'); }

  // 数据操作
  async insert(tableName, data) { throw new Error('Must implement insert'); }
  async update(tableName, criteria, data) { throw new Error('Must implement update'); }
  async delete(tableName, criteria) { throw new Error('Must implement delete'); }
  async select(tableName, options) { throw new Error('Must implement select'); }
}
```

### 2. PostgreSQL 适配器

```javascript
// @strapi/database/lib/adapters/postgresql.js
class PostgreSQLAdapter extends DatabaseAdapter {
  constructor(config) {
    super(config);
    this.client = knex({
      client: 'pg',
      connection: config.connection,
      pool: config.pool,
      acquireConnectionTimeout: config.acquireConnectionTimeout
    });
  }

  async executeQuery(sql, params = []) {
    try {
      const result = await this.client.raw(sql, params);
      return result.rows || result;
    } catch (error) {
      throw new DatabaseError(error.message, {
        sql,
        params,
        originalError: error
      });
    }
  }

  async transaction(callback) {
    return this.client.transaction(async (trx) => {
      const transactionalAdapter = new PostgreSQLAdapter({
        ...this.config,
        client: trx
      });
      return callback(transactionalAdapter);
    });
  }

  // PostgreSQL特有的功能
  async createIndex(tableName, indexName, columns, options = {}) {
    const indexType = options.type || 'btree';
    const unique = options.unique ? 'UNIQUE' : '';

    const sql = `
      CREATE ${unique} INDEX ${indexName}
      ON ${tableName} USING ${indexType} (${columns.join(', ')})
    `;

    return this.executeQuery(sql);
  }

  // JSON字段支持
  async queryJsonField(tableName, jsonColumn, jsonPath, value) {
    const sql = `
      SELECT * FROM ${tableName}
      WHERE ${jsonColumn}->>'${jsonPath}' = ?
    `;
    return this.executeQuery(sql, [value]);
  }
}
```

### 3. MySQL 适配器

```javascript
// @strapi/database/lib/adapters/mysql.js
class MySQLAdapter extends DatabaseAdapter {
  constructor(config) {
    super(config);
    this.client = knex({
      client: 'mysql2',
      connection: {
        ...config.connection,
        timezone: 'UTC',
        charset: 'utf8mb4',
        collation: 'utf8mb4_unicode_ci'
      }
    });
  }

  // MySQL特有的功能
  async createFullTextIndex(tableName, columns) {
    const sql = `
      ALTER TABLE ${tableName}
      ADD FULLTEXT INDEX ft_idx (${columns.join(', ')})
    `;
    return this.executeQuery(sql);
  }

  async fullTextSearch(tableName, columns, searchTerm) {
    const columnList = columns.join(', ');
    const sql = `
      SELECT *, MATCH(${columnList}) AGAINST(? IN NATURAL LANGUAGE MODE) as score
      FROM ${tableName}
      WHERE MATCH(${columnList}) AGAINST(? IN NATURAL LANGUAGE MODE)
      ORDER BY score DESC
    `;
    return this.executeQuery(sql, [searchTerm, searchTerm]);
  }
}
```

## 查询构建器系统

### 1. 核心查询构建器

```javascript
// @strapi/database/lib/query-builder.js
class QueryBuilder {
  constructor(database) {
    this.database = database;
    this.model = null;
    this.queryState = {
      select: [],
      where: [],
      joins: [],
      orderBy: [],
      groupBy: [],
      having: [],
      limit: null,
      offset: null,
      populate: []
    };
  }

  // 链式方法
  select(fields) {
    if (Array.isArray(fields)) {
      this.queryState.select.push(...fields);
    } else {
      this.queryState.select.push(fields);
    }
    return this;
  }

  where(field, operator, value) {
    if (typeof field === 'object') {
      // 处理对象形式的where条件
      Object.entries(field).forEach(([key, val]) => {
        this.queryState.where.push({ field: key, operator: '=', value: val });
      });
    } else {
      this.queryState.where.push({ field, operator, value });
    }
    return this;
  }

  // 复杂过滤器支持
  filters(filterObj) {
    this.parseFilters(filterObj, this.queryState.where);
    return this;
  }

  parseFilters(filters, whereClause, prefix = '') {
    Object.entries(filters).forEach(([key, value]) => {
      const fieldPath = prefix ? `${prefix}.${key}` : key;

      if (key.startsWith('$')) {
        // 处理操作符
        this.handleOperator(key, value, whereClause, prefix);
      } else if (typeof value === 'object' && value !== null) {
        // 递归处理嵌套过滤器
        this.parseFilters(value, whereClause, fieldPath);
      } else {
        // 简单相等比较
        whereClause.push({ field: fieldPath, operator: '=', value });
      }
    });
  }

  handleOperator(operator, value, whereClause, fieldPath) {
    switch (operator) {
      case '$eq':
        whereClause.push({ field: fieldPath, operator: '=', value });
        break;
      case '$ne':
        whereClause.push({ field: fieldPath, operator: '!=', value });
        break;
      case '$gt':
        whereClause.push({ field: fieldPath, operator: '>', value });
        break;
      case '$gte':
        whereClause.push({ field: fieldPath, operator: '>=', value });
        break;
      case '$lt':
        whereClause.push({ field: fieldPath, operator: '<', value });
        break;
      case '$lte':
        whereClause.push({ field: fieldPath, operator: '<=', value });
        break;
      case '$in':
        whereClause.push({ field: fieldPath, operator: 'IN', value });
        break;
      case '$notIn':
        whereClause.push({ field: fieldPath, operator: 'NOT IN', value });
        break;
      case '$contains':
        whereClause.push({ field: fieldPath, operator: 'LIKE', value: `%${value}%` });
        break;
      case '$startsWith':
        whereClause.push({ field: fieldPath, operator: 'LIKE', value: `${value}%` });
        break;
      case '$endsWith':
        whereClause.push({ field: fieldPath, operator: 'LIKE', value: `%${value}` });
        break;
      case '$null':
        const nullOp = value ? 'IS NULL' : 'IS NOT NULL';
        whereClause.push({ field: fieldPath, operator: nullOp, value: null });
        break;
    }
  }

  // 关系查询
  populate(relations) {
    if (typeof relations === 'string') {
      this.queryState.populate.push(relations);
    } else if (Array.isArray(relations)) {
      this.queryState.populate.push(...relations);
    } else if (typeof relations === 'object') {
      // 处理复杂的populate配置
      Object.entries(relations).forEach(([relation, config]) => {
        this.queryState.populate.push({ relation, config });
      });
    }
    return this;
  }

  // 排序
  orderBy(field, direction = 'ASC') {
    this.queryState.orderBy.push({ field, direction: direction.toUpperCase() });
    return this;
  }

  // 分页
  limit(count) {
    this.queryState.limit = count;
    return this;
  }

  offset(count) {
    this.queryState.offset = count;
    return this;
  }

  // 执行查询
  async execute() {
    const sql = this.buildSQL();
    const params = this.buildParams();
    const result = await this.database.executeQuery(sql, params);

    // 处理关系数据
    if (this.queryState.populate.length > 0) {
      return this.populateRelations(result);
    }

    return result;
  }

  buildSQL() {
    let sql = 'SELECT ';

    // SELECT子句
    if (this.queryState.select.length > 0) {
      sql += this.queryState.select.join(', ');
    } else {
      sql += '*';
    }

    // FROM子句
    sql += ` FROM ${this.model.tableName}`;

    // JOIN子句
    if (this.queryState.joins.length > 0) {
      sql += ' ' + this.queryState.joins.map(join =>
        `${join.type} JOIN ${join.table} ON ${join.condition}`
      ).join(' ');
    }

    // WHERE子句
    if (this.queryState.where.length > 0) {
      sql += ' WHERE ' + this.queryState.where.map(where =>
        `${where.field} ${where.operator} ?`
      ).join(' AND ');
    }

    // GROUP BY子句
    if (this.queryState.groupBy.length > 0) {
      sql += ' GROUP BY ' + this.queryState.groupBy.join(', ');
    }

    // HAVING子句
    if (this.queryState.having.length > 0) {
      sql += ' HAVING ' + this.queryState.having.map(having =>
        `${having.field} ${having.operator} ?`
      ).join(' AND ');
    }

    // ORDER BY子句
    if (this.queryState.orderBy.length > 0) {
      sql += ' ORDER BY ' + this.queryState.orderBy.map(order =>
        `${order.field} ${order.direction}`
      ).join(', ');
    }

    // LIMIT和OFFSET
    if (this.queryState.limit !== null) {
      sql += ` LIMIT ${this.queryState.limit}`;
    }

    if (this.queryState.offset !== null) {
      sql += ` OFFSET ${this.queryState.offset}`;
    }

    return sql;
  }

  buildParams() {
    const params = [];

    // WHERE参数
    this.queryState.where.forEach(where => {
      if (where.value !== null) {
        params.push(where.value);
      }
    });

    // HAVING参数
    this.queryState.having.forEach(having => {
      if (having.value !== null) {
        params.push(having.value);
      }
    });

    return params;
  }
}
```

## Entity Service 层

### 1. 实体服务接口

```javascript
// @strapi/database/lib/entity-service.js
class EntityService {
  constructor(database) {
    this.database = database;
  }

  // 查找多个实体
  async findMany(uid, params = {}) {
    const model = this.database.metadata.get(uid);
    const qb = this.database.queryBuilder
      .model(model)
      .filters(params.filters || {})
      .populate(params.populate || []);

    // 处理排序
    if (params.sort) {
      if (Array.isArray(params.sort)) {
        params.sort.forEach(sortItem => {
          if (typeof sortItem === 'string') {
            const [field, direction] = sortItem.split(':');
            qb.orderBy(field, direction || 'ASC');
          }
        });
      }
    }

    // 处理分页
    if (params.pagination) {
      const { page = 1, pageSize = 25 } = params.pagination;
      qb.limit(pageSize).offset((page - 1) * pageSize);
    }

    const results = await qb.execute();

    // 如果需要分页信息
    if (params.pagination) {
      const total = await this.count(uid, { filters: params.filters });
      const { page = 1, pageSize = 25 } = params.pagination;

      return {
        results,
        pagination: {
          page,
          pageSize,
          pageCount: Math.ceil(total / pageSize),
          total
        }
      };
    }

    return results;
  }

  // 查找单个实体
  async findOne(uid, id, params = {}) {
    const model = this.database.metadata.get(uid);
    const qb = this.database.queryBuilder
      .model(model)
      .where('id', '=', id)
      .populate(params.populate || []);

    const results = await qb.execute();
    return results[0] || null;
  }

  // 创建实体
  async create(uid, data, params = {}) {
    const model = this.database.metadata.get(uid);

    // 数据验证
    const validatedData = await this.validateData(model, data);

    // 处理关系数据
    const { mainData, relationData } = this.separateRelationData(model, validatedData);

    return this.database.transaction(async (trx) => {
      // 插入主数据
      const [id] = await trx.insert(model.tableName, {
        ...mainData,
        created_at: new Date(),
        updated_at: new Date()
      });

      // 处理关系
      await this.handleRelations(trx, model, id, relationData);

      // 返回创建的实体
      return this.findOne(uid, id, params);
    });
  }

  // 更新实体
  async update(uid, id, data, params = {}) {
    const model = this.database.metadata.get(uid);
    const validatedData = await this.validateData(model, data);
    const { mainData, relationData } = this.separateRelationData(model, validatedData);

    return this.database.transaction(async (trx) => {
      // 更新主数据
      await trx.update(model.tableName, { id }, {
        ...mainData,
        updated_at: new Date()
      });

      // 处理关系更新
      await this.handleRelations(trx, model, id, relationData);

      // 返回更新后的实体
      return this.findOne(uid, id, params);
    });
  }

  // 删除实体
  async delete(uid, id) {
    const model = this.database.metadata.get(uid);

    return this.database.transaction(async (trx) => {
      // 获取要删除的实体
      const entity = await this.findOne(uid, id);
      if (!entity) {
        throw new Error(`Entity with id ${id} not found`);
      }

      // 清理关系
      await this.cleanupRelations(trx, model, id);

      // 删除主记录
      await trx.delete(model.tableName, { id });

      return entity;
    });
  }

  // 计数
  async count(uid, params = {}) {
    const model = this.database.metadata.get(uid);
    const qb = this.database.queryBuilder
      .model(model)
      .filters(params.filters || {})
      .select('COUNT(*) as count');

    const result = await qb.execute();
    return result[0].count;
  }

  // 数据验证
  async validateData(model, data) {
    const validatedData = {};

    for (const [fieldName, fieldConfig] of Object.entries(model.attributes)) {
      const value = data[fieldName];

      if (value !== undefined) {
        // 类型验证
        const validatedValue = await this.validateField(fieldConfig, value);
        validatedData[fieldName] = validatedValue;
      }
    }

    return validatedData;
  }

  async validateField(fieldConfig, value) {
    switch (fieldConfig.type) {
      case 'string':
        if (typeof value !== 'string') {
          throw new ValidationError(`Expected string, got ${typeof value}`);
        }
        if (fieldConfig.maxLength && value.length > fieldConfig.maxLength) {
          throw new ValidationError(`String too long: ${value.length} > ${fieldConfig.maxLength}`);
        }
        return value;

      case 'integer':
        const intValue = parseInt(value, 10);
        if (isNaN(intValue)) {
          throw new ValidationError(`Invalid integer: ${value}`);
        }
        return intValue;

      case 'float':
        const floatValue = parseFloat(value);
        if (isNaN(floatValue)) {
          throw new ValidationError(`Invalid float: ${value}`);
        }
        return floatValue;

      case 'boolean':
        return Boolean(value);

      case 'datetime':
        return new Date(value);

      default:
        return value;
    }
  }
}
```

## 关系管理系统

### 1. 关系类型定义

```javascript
// @strapi/database/lib/relations.js
class RelationManager {
  constructor(database) {
    this.database = database;
  }

  // 处理一对一关系
  async handleOneToOne(model, id, relationName, relationData) {
    const relation = model.attributes[relationName];
    const targetModel = this.database.metadata.get(relation.target);

    if (relationData.connect) {
      // 连接现有实体
      await this.database.query(model.tableName)
        .update({ id }, { [relation.foreignKey]: relationData.connect.id });
    } else if (relationData.create) {
      // 创建新实体
      const createdEntity = await this.database.entityService.create(
        relation.target,
        relationData.create
      );
      await this.database.query(model.tableName)
        .update({ id }, { [relation.foreignKey]: createdEntity.id });
    } else if (relationData.disconnect) {
      // 断开连接
      await this.database.query(model.tableName)
        .update({ id }, { [relation.foreignKey]: null });
    }
  }

  // 处理一对多关系
  async handleOneToMany(model, id, relationName, relationData) {
    const relation = model.attributes[relationName];
    const targetModel = this.database.metadata.get(relation.target);

    if (relationData.connect) {
      // 连接现有实体
      const connectIds = Array.isArray(relationData.connect)
        ? relationData.connect.map(item => item.id)
        : [relationData.connect.id];

      await this.database.query(targetModel.tableName)
        .whereIn('id', connectIds)
        .update({ [relation.inversedBy]: id });
    }

    if (relationData.disconnect) {
      // 断开连接
      const disconnectIds = Array.isArray(relationData.disconnect)
        ? relationData.disconnect.map(item => item.id)
        : [relationData.disconnect.id];

      await this.database.query(targetModel.tableName)
        .whereIn('id', disconnectIds)
        .update({ [relation.inversedBy]: null });
    }

    if (relationData.create) {
      // 创建新实体
      const createData = Array.isArray(relationData.create)
        ? relationData.create
        : [relationData.create];

      for (const data of createData) {
        await this.database.entityService.create(relation.target, {
          ...data,
          [relation.inversedBy]: id
        });
      }
    }
  }

  // 处理多对多关系
  async handleManyToMany(model, id, relationName, relationData) {
    const relation = model.attributes[relationName];
    const joinTable = relation.joinTable || `${model.tableName}_${relationName}`;
    const joinColumn = relation.joinColumn || `${model.tableName}_id`;
    const inverseJoinColumn = relation.inverseJoinColumn || `${relation.target}_id`;

    if (relationData.connect) {
      // 连接现有实体
      const connectIds = Array.isArray(relationData.connect)
        ? relationData.connect.map(item => item.id)
        : [relationData.connect.id];

      for (const targetId of connectIds) {
        await this.database.query(joinTable).insert({
          [joinColumn]: id,
          [inverseJoinColumn]: targetId
        });
      }
    }

    if (relationData.disconnect) {
      // 断开连接
      const disconnectIds = Array.isArray(relationData.disconnect)
        ? relationData.disconnect.map(item => item.id)
        : [relationData.disconnect.id];

      await this.database.query(joinTable)
        .where(joinColumn, id)
        .whereIn(inverseJoinColumn, disconnectIds)
        .delete();
    }

    if (relationData.set) {
      // 设置关系（先清空再设置）
      await this.database.query(joinTable)
        .where(joinColumn, id)
        .delete();

      const setIds = Array.isArray(relationData.set)
        ? relationData.set.map(item => item.id)
        : [relationData.set.id];

      for (const targetId of setIds) {
        await this.database.query(joinTable).insert({
          [joinColumn]: id,
          [inverseJoinColumn]: targetId
        });
      }
    }
  }

  // 加载关系数据
  async populateRelations(entities, populate, model) {
    if (!Array.isArray(entities)) {
      entities = [entities];
    }

    for (const relationName of populate) {
      const relation = model.attributes[relationName];
      if (!relation) continue;

      switch (relation.relation) {
        case 'oneToOne':
        case 'manyToOne':
          await this.populateToOneRelation(entities, relationName, relation);
          break;

        case 'oneToMany':
          await this.populateOneToManyRelation(entities, relationName, relation);
          break;

        case 'manyToMany':
          await this.populateManyToManyRelation(entities, relationName, relation);
          break;
      }
    }

    return entities;
  }

  async populateToOneRelation(entities, relationName, relation) {
    const foreignKeys = entities.map(entity => entity[relation.foreignKey]).filter(Boolean);
    if (foreignKeys.length === 0) return;

    const relatedEntities = await this.database.entityService.findMany(relation.target, {
      filters: { id: { $in: foreignKeys } }
    });

    const relatedMap = new Map(relatedEntities.map(entity => [entity.id, entity]));

    entities.forEach(entity => {
      const foreignKey = entity[relation.foreignKey];
      entity[relationName] = foreignKey ? relatedMap.get(foreignKey) : null;
    });
  }

  async populateOneToManyRelation(entities, relationName, relation) {
    const entityIds = entities.map(entity => entity.id);

    const relatedEntities = await this.database.entityService.findMany(relation.target, {
      filters: { [relation.inversedBy]: { $in: entityIds } }
    });

    const relatedMap = new Map();
    relatedEntities.forEach(entity => {
      const ownerId = entity[relation.inversedBy];
      if (!relatedMap.has(ownerId)) {
        relatedMap.set(ownerId, []);
      }
      relatedMap.get(ownerId).push(entity);
    });

    entities.forEach(entity => {
      entity[relationName] = relatedMap.get(entity.id) || [];
    });
  }

  async populateManyToManyRelation(entities, relationName, relation) {
    const joinTable = relation.joinTable || `${model.tableName}_${relationName}`;
    const joinColumn = relation.joinColumn || `${model.tableName}_id`;
    const inverseJoinColumn = relation.inverseJoinColumn || `${relation.target}_id`;

    const entityIds = entities.map(entity => entity.id);

    // 获取连接表数据
    const joinRecords = await this.database.query(joinTable)
      .whereIn(joinColumn, entityIds)
      .select();

    // 获取关联实体IDs
    const relatedIds = [...new Set(joinRecords.map(record => record[inverseJoinColumn]))];

    // 获取关联实体
    const relatedEntities = await this.database.entityService.findMany(relation.target, {
      filters: { id: { $in: relatedIds } }
    });

    const relatedMap = new Map(relatedEntities.map(entity => [entity.id, entity]));
    const entityRelationMap = new Map();

    joinRecords.forEach(record => {
      const entityId = record[joinColumn];
      const relatedId = record[inverseJoinColumn];

      if (!entityRelationMap.has(entityId)) {
        entityRelationMap.set(entityId, []);
      }

      const relatedEntity = relatedMap.get(relatedId);
      if (relatedEntity) {
        entityRelationMap.get(entityId).push(relatedEntity);
      }
    });

    entities.forEach(entity => {
      entity[relationName] = entityRelationMap.get(entity.id) || [];
    });
  }
}
```

## 迁移系统

### 1. 迁移管理器

```javascript
// @strapi/database/lib/migrations/migration-manager.js
class MigrationManager {
  constructor(database) {
    this.database = database;
    this.migrationsTable = 'strapi_database_schema';
  }

  async initialize() {
    // 创建迁移表
    await this.createMigrationsTable();
  }

  async createMigrationsTable() {
    const exists = await this.database.adapter.tableExists(this.migrationsTable);
    if (!exists) {
      await this.database.adapter.createTable(this.migrationsTable, {
        id: { type: 'integer', primaryKey: true, autoIncrement: true },
        name: { type: 'string', notNull: true },
        time: { type: 'datetime', notNull: true },
        hash: { type: 'string', notNull: true }
      });
    }
  }

  async runMigrations() {
    const pendingMigrations = await this.getPendingMigrations();

    for (const migration of pendingMigrations) {
      console.log(`Running migration: ${migration.name}`);

      await this.database.transaction(async (trx) => {
        try {
          await migration.up(trx);
          await this.recordMigration(migration);
          console.log(`✓ Migration ${migration.name} completed`);
        } catch (error) {
          console.error(`✗ Migration ${migration.name} failed:`, error);
          throw error;
        }
      });
    }
  }

  async getPendingMigrations() {
    // 获取已执行的迁移
    const executedMigrations = await this.database.query(this.migrationsTable)
      .select('name')
      .execute();

    const executedNames = new Set(executedMigrations.map(m => m.name));

    // 获取所有迁移文件
    const allMigrations = await this.loadMigrationFiles();

    // 过滤出待执行的迁移
    return allMigrations.filter(migration => !executedNames.has(migration.name));
  }

  async recordMigration(migration) {
    await this.database.query(this.migrationsTable).insert({
      name: migration.name,
      time: new Date(),
      hash: migration.hash
    });
  }
}
```

### 2. Schema 迁移

```javascript
// 示例迁移文件
const migration = {
  name: '2024-01-01-create-articles-table',
  hash: 'abc123',

  async up(database) {
    // 创建文章表
    await database.createTable('articles', {
      id: {
        type: 'integer',
        primaryKey: true,
        autoIncrement: true
      },
      title: {
        type: 'string',
        notNull: true,
        maxLength: 255
      },
      content: {
        type: 'text'
      },
      published_at: {
        type: 'datetime'
      },
      created_at: {
        type: 'datetime',
        notNull: true,
        defaultTo: database.fn.now()
      },
      updated_at: {
        type: 'datetime',
        notNull: true,
        defaultTo: database.fn.now()
      }
    });

    // 创建索引
    await database.createIndex('articles', 'idx_articles_title', ['title']);
    await database.createIndex('articles', 'idx_articles_published_at', ['published_at']);

    // 创建关系表
    await database.createTable('articles_authors', {
      id: {
        type: 'integer',
        primaryKey: true,
        autoIncrement: true
      },
      article_id: {
        type: 'integer',
        notNull: true,
        references: { table: 'articles', column: 'id' },
        onDelete: 'CASCADE'
      },
      author_id: {
        type: 'integer',
        notNull: true,
        references: { table: 'users', column: 'id' },
        onDelete: 'CASCADE'
      }
    });

    // 创建唯一约束
    await database.createIndex('articles_authors', 'idx_articles_authors_unique',
      ['article_id', 'author_id'], { unique: true });
  },

  async down(database) {
    await database.dropTable('articles_authors');
    await database.dropTable('articles');
  }
};
```

## 性能优化

### 1. 连接池管理

```javascript
class ConnectionPool {
  constructor(config) {
    this.config = {
      min: config.pool?.min || 2,
      max: config.pool?.max || 10,
      acquireTimeoutMillis: config.pool?.acquireTimeoutMillis || 30000,
      createTimeoutMillis: config.pool?.createTimeoutMillis || 30000,
      destroyTimeoutMillis: config.pool?.destroyTimeoutMillis || 5000,
      idleTimeoutMillis: config.pool?.idleTimeoutMillis || 30000,
      reapIntervalMillis: config.pool?.reapIntervalMillis || 1000,
      createRetryIntervalMillis: config.pool?.createRetryIntervalMillis || 200,
      ...config.pool
    };
  }

  initialize() {
    this.pool = knex({
      client: this.config.client,
      connection: this.config.connection,
      pool: this.config,
      acquireConnectionTimeout: this.config.acquireTimeoutMillis
    });
  }

  async getConnection() {
    return this.pool.client.acquireConnection();
  }

  async releaseConnection(connection) {
    return this.pool.client.releaseConnection(connection);
  }

  async destroy() {
    if (this.pool) {
      await this.pool.destroy();
    }
  }
}
```

### 2. 查询缓存

```javascript
class QueryCache {
  constructor(config = {}) {
    this.enabled = config.enabled !== false;
    this.ttl = config.ttl || 300000; // 5分钟默认TTL
    this.maxSize = config.maxSize || 1000;
    this.cache = new Map();
    this.timers = new Map();
  }

  generateCacheKey(sql, params) {
    return `${sql}:${JSON.stringify(params)}`;
  }

  get(sql, params) {
    if (!this.enabled) return null;

    const key = this.generateCacheKey(sql, params);
    return this.cache.get(key);
  }

  set(sql, params, result) {
    if (!this.enabled) return;

    const key = this.generateCacheKey(sql, params);

    // 检查缓存大小限制
    if (this.cache.size >= this.maxSize) {
      this.evictOldest();
    }

    this.cache.set(key, {
      result,
      timestamp: Date.now()
    });

    // 设置过期定时器
    const timer = setTimeout(() => {
      this.cache.delete(key);
      this.timers.delete(key);
    }, this.ttl);

    this.timers.set(key, timer);
  }

  evictOldest() {
    const oldestKey = this.cache.keys().next().value;
    if (oldestKey) {
      this.cache.delete(oldestKey);
      const timer = this.timers.get(oldestKey);
      if (timer) {
        clearTimeout(timer);
        this.timers.delete(oldestKey);
      }
    }
  }

  clear() {
    this.cache.clear();
    this.timers.forEach(timer => clearTimeout(timer));
    this.timers.clear();
  }
}
```

### 3. 批量操作优化

```javascript
class BatchOperations {
  constructor(database) {
    this.database = database;
  }

  async batchInsert(tableName, records, batchSize = 100) {
    const batches = [];
    for (let i = 0; i < records.length; i += batchSize) {
      batches.push(records.slice(i, i + batchSize));
    }

    const results = [];
    for (const batch of batches) {
      const batchResult = await this.database.adapter.batchInsert(tableName, batch);
      results.push(...batchResult);
    }

    return results;
  }

  async batchUpdate(tableName, updates, batchSize = 100) {
    const batches = [];
    for (let i = 0; i < updates.length; i += batchSize) {
      batches.push(updates.slice(i, i + batchSize));
    }

    for (const batch of batches) {
      await this.database.transaction(async (trx) => {
        for (const update of batch) {
          await trx.update(tableName, update.criteria, update.data);
        }
      });
    }
  }

  async batchDelete(tableName, criteria, batchSize = 100) {
    if (Array.isArray(criteria)) {
      const batches = [];
      for (let i = 0; i < criteria.length; i += batchSize) {
        batches.push(criteria.slice(i, i + batchSize));
      }

      for (const batch of batches) {
        await this.database.query(tableName)
          .whereIn('id', batch.map(c => c.id))
          .delete();
      }
    } else {
      await this.database.query(tableName)
        .where(criteria)
        .delete();
    }
  }
}
```

## 事务管理

### 1. 事务接口

```javascript
class TransactionManager {
  constructor(database) {
    this.database = database;
  }

  async transaction(callback, options = {}) {
    const isolation = options.isolation || 'READ COMMITTED';

    return this.database.adapter.transaction(async (trx) => {
      // 设置事务隔离级别
      await trx.raw(`SET TRANSACTION ISOLATION LEVEL ${isolation}`);

      // 创建事务上下文
      const transactionContext = {
        database: this.database,
        adapter: trx,
        query: (tableName) => new QueryBuilder(this.database).transaction(trx).model(tableName),
        entityService: new EntityService(this.database).transaction(trx)
      };

      try {
        const result = await callback(transactionContext);
        await trx.commit();
        return result;
      } catch (error) {
        await trx.rollback();
        throw error;
      }
    });
  }

  async savepoint(name, callback) {
    return this.database.adapter.savepoint(name, callback);
  }
}
```

## 总结

Strapi的数据库抽象层通过分层架构、适配器模式、查询构建器等设计模式，实现了：

### 优势
1. **多数据库支持**: 统一接口适配不同数据库
2. **类型安全**: TypeScript支持和运行时验证
3. **性能优化**: 连接池、查询缓存、批量操作
4. **关系管理**: 复杂的关系映射和lazy loading
5. **事务支持**: 完整的ACID事务管理
6. **迁移系统**: 自动化的schema管理

### 设计特点
- **模块化**: 每个组件职责单一，易于测试和维护
- **可扩展**: 插件可以扩展数据库功能
- **现代化**: 支持Promise/async-await、链式调用
- **开发友好**: 直观的API和丰富的错误信息

这种设计使得Strapi能够在保持高性能的同时，提供简洁易用的数据访问接口，满足现代Web应用的复杂数据需求。
