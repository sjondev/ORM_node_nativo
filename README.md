# ORM_node_nativo

```
import { pool } from '../database/database';
import type { PoolClient } from 'pg';

// ====================== TIPAGENS AVANÇADAS ======================

export type WhereOperators<FieldType> = {
  $gt?: FieldType;
  $gte?: FieldType;
  $lt?: FieldType;
  $lte?: FieldType;
  $ne?: FieldType;
  $like?: string;
  $ilike?: string;
  $in?: FieldType[];
  $nin?: FieldType[];
  $between?: [FieldType, FieldType];
  $notBetween?: [FieldType, FieldType];
  $isNull?: boolean;
  $isNotNull?: boolean;
};

// Mágica para permitir $or com as chaves corretas
export type WhereFilter<T> = {
  [K in keyof T]?: T[K] | null | WhereOperators<T[K]>;
} & {
  $or?: Array<WhereFilter<T>>;
};

export interface FindOptions<T> {
  select?: Array<keyof T>;
  /** Exclui as colunas passadas (MUITO útil para esconder senhas) */
  exclude?: Array<keyof T>;
  where?: WhereFilter<T>;
  orderBy?: Array<{ column: keyof T; direction?: 'ASC' | 'DESC' }>;
  limit?: number;
  offset?: number;
  tx?: PoolClient;
  withDeleted?: boolean;
}

type FindOneOptions<T> = Omit<FindOptions<T>, 'limit' | 'offset'>;

// ====================== CLASSE BASE (O FRAMEWORK SUPREMO) ======================
export abstract class BaseModel<T> {
  protected abstract tableName: string;
  protected primaryKey: string = 'id';

  protected useTimestamps: boolean = false;
  protected useSoftDeletes: boolean = false;
  protected debugLog: boolean = false;

  // Cache interno para otimização de queries com exclude
  private cachedColumns: Array<keyof T> | null = null;

  // --- HOOKS ---
  protected async beforeInsert(data: Partial<T>): Promise<Partial<T>> {
    if (this.useTimestamps) {
      return { ...data, created_at: new Date(), updated_at: new Date() } as Partial<T>;
    }
    return data;
  }

  protected async beforeUpdate(data: Partial<T>): Promise<Partial<T>> {
    if (this.useTimestamps) {
      return { ...data, updated_at: new Date() } as Partial<T>;
    }
    return data;
  }

  // --- INFRAESTRUTURA ---
  protected getClient(tx?: PoolClient) {
    return tx || pool;
  }

  private logQuery(query: string, params: any[]) {
    if (this.debugLog) {
      console.log(`\n[ORM DEBUG - ${this.tableName}]\nQuery: ${query}\nParams:`, params, '\n');
    }
  }

  private escapeId(identifier: string | number | symbol): string {
    return String(identifier).split('.').map(p => `"${p}"`).join('.');
  }

  // Método que busca as colunas da tabela na primeira vez e guarda na memória (Cache)
  private async getTableColumns(tx?: PoolClient): Promise<Array<keyof T>> {
    if (this.cachedColumns) return this.cachedColumns;

    const query = `
      SELECT column_name 
      FROM information_schema.columns 
      WHERE table_name = $1
    `;
    const client = this.getClient(tx);
    const { rows } = await client.query(query, [this.tableName]);
    
    this.cachedColumns = rows.map(r => r.column_name as keyof T);
    return this.cachedColumns;
  }

  // --- MOTOR DE WHERE RECURSIVO ---
  private buildWhereRecursive(conditions: any, params: any[]): string {
    if (!conditions) return '';
    
    const entries = Object.entries(conditions).filter(([, v]) => v !== undefined);
    if (entries.length === 0) return '';

    const whereParts: string[] = [];

    for (const [key, value] of entries) {
      // Motor Avançado de $or
      if (key === '$or' && Array.isArray(value)) {
        const orParts: string[] = [];
        for (const orCondition of value) {
          const subClause = this.buildWhereRecursive(orCondition, params);
          if (subClause) orParts.push(`(${subClause})`);
        }
        if (orParts.length > 0) {
          whereParts.push(`(${orParts.join(' OR ')})`);
        }
        continue;
      }

      const escapedKey = this.escapeId(key);

      if (value === null) {
        whereParts.push(`${escapedKey} IS NULL`);
        continue;
      }

      if (Array.isArray(value)) {
        if (value.length === 0) {
          whereParts.push('1 = 0');
        } else {
          const placeholders = value.map(v => {
            params.push(v);
            return `$${params.length}`;
          }).join(', ');
          whereParts.push(`${escapedKey} IN (${placeholders})`);
        }
        continue;
      }

      if (value && typeof value === 'object' && !Array.isArray(value) && !(value instanceof Date)) {
        const operators = Object.entries(value);
        
        for (const [op, opValue] of operators) {
          switch (op) {
            case '$gt': case '$gte': case '$lt': case '$lte': case '$ne': {
              const symbol = { $gt: '>', $gte: '>=', $lt: '<', $lte: '<=', $ne: '!=' }[op];
              params.push(opValue);
              whereParts.push(`${escapedKey} ${symbol} $${params.length}`);
              break;
            }
            case '$like': {
              params.push(opValue);
              whereParts.push(`${escapedKey} LIKE $${params.length}`);
              break;
            }
            case '$ilike': {
              params.push(opValue);
              whereParts.push(`${escapedKey} ILIKE $${params.length}`);
              break;
            }
            case '$in':
            case '$nin': {
              const inArr = opValue as any[];
              if (inArr.length === 0) {
                whereParts.push('1 = 0');
              } else {
                const operator = op === '$nin' ? 'NOT IN' : 'IN';
                const ph = inArr.map(v => {
                  params.push(v);
                  return `$${params.length}`;
                }).join(', ');
                whereParts.push(`${escapedKey} ${operator} (${ph})`);
              }
              break;
            }
            case '$between':
            case '$notBetween': {
              const arr = opValue as any[];
              if (arr.length !== 2) throw new Error(`Operador ${op} requer array com 2 valores`);
              const opStr = op === '$notBetween' ? 'NOT BETWEEN' : 'BETWEEN';
              params.push(arr[0], arr[1]);
              whereParts.push(`${escapedKey} ${opStr} $${params.length - 1} AND $${params.length}`);
              break;
            }
            case '$isNull': {
              if (opValue) whereParts.push(`${escapedKey} IS NULL`);
              break;
            }
            case '$isNotNull': {
              if (opValue) whereParts.push(`${escapedKey} IS NOT NULL`);
              break;
            }
            default:
              throw new Error(`Operador não suportado: ${op}`);
          }
        }
        continue;
      }

      params.push(value);
      whereParts.push(`${escapedKey} = $${params.length}`);
    }

    return whereParts.join(' AND ');
  }

  private buildWhere(conditions: any = {}, withDeleted = false, startingParams: any[] = []): { whereClause: string; params: any[] } {
    const finalConditions = { ...conditions };
    const params = [...startingParams];

    if (this.useSoftDeletes && !withDeleted && finalConditions['deleted_at'] === undefined) {
      finalConditions['deleted_at'] = { $isNull: true }; // Usando o novo operador para proteção!
    }

    const logicString = this.buildWhereRecursive(finalConditions, params);
    
    return {
      whereClause: logicString ? `WHERE ${logicString}` : '',
      params,
    };
  }

  // --- MÉTODOS PÚBLICOS DE CONSULTA ---
  async find(options: FindOptions<T> = {}): Promise<T[]> {
    const { whereClause, params } = this.buildWhere(options.where, options.withDeleted);

    let selectClause = '*';
    
    // Tratativa poderosa: Exclude e Select!
    if (options.exclude && options.exclude.length > 0) {
      const allColumns = await this.getTableColumns(options.tx);
      const wantedColumns = allColumns.filter(col => !options.exclude!.includes(col));
      selectClause = wantedColumns.map(c => this.escapeId(c)).join(', ');
    } else if (options.select && options.select.length > 0) {
      selectClause = options.select.map(c => this.escapeId(c)).join(', ');
    }

    const queryParts: string[] = [`SELECT ${selectClause} FROM ${this.escapeId(this.tableName)}`];
    if (whereClause) queryParts.push(whereClause);

    if (options.orderBy?.length) {
      const orderParts = options.orderBy
        .map(({ column, direction = 'ASC' }) => `${this.escapeId(column)} ${direction}`)
        .join(', ');
      queryParts.push(`ORDER BY ${orderParts}`);
    }

    if (options.limit !== undefined) {
      params.push(options.limit);
      queryParts.push(`LIMIT $${params.length}`);
    }
    if (options.offset !== undefined) {
      params.push(options.offset);
      queryParts.push(`OFFSET $${params.length}`);
    }

    const query = queryParts.join(' ');
    this.logQuery(query, params);

    const client = this.getClient(options.tx);
    const { rows } = await client.query(query, params);
    return rows;
  }

  async findOne(options: FindOneOptions<T> = {}): Promise<T | undefined> {
    const results = await this.find({ ...options, limit: 1 });
    return results[0];
  }

  async findById(id: number | string, options?: { select?: Array<keyof T>; exclude?: Array<keyof T>; tx?: PoolClient; withDeleted?: boolean }): Promise<T | undefined> {
    return this.findOne({
      where: { [this.primaryKey]: id } as any,
      select: options?.select,
      exclude: options?.exclude,
      tx: options?.tx,
      withDeleted: options?.withDeleted
    });
  }

  async exists(where: FindOptions<T>['where'] = {}, tx?: PoolClient, withDeleted = false): Promise<boolean> {
    const { whereClause, params } = this.buildWhere(where, withDeleted);
    const query = `SELECT 1 FROM ${this.escapeId(this.tableName)} ${whereClause} LIMIT 1`;
    const client = this.getClient(tx);
    const { rowCount } = await client.query(query, params);
    return (rowCount ?? 0) > 0;
  }

  async count(where?: FindOptions<T>['where'], tx?: PoolClient, withDeleted = false): Promise<number> {
    const { whereClause, params } = this.buildWhere(where ?? {}, withDeleted);
    const query = `SELECT COUNT(*) as count FROM ${this.escapeId(this.tableName)} ${whereClause}`;
    const client = this.getClient(tx);
    const { rows } = await client.query(query, params);
    return parseInt(rows[0]?.count || '0', 10);
  }

  async paginate(page: number, limit: number, options: Omit<FindOptions<T>, 'limit' | 'offset'> = {}) {
    const offset = (page - 1) * limit;
    const [data, total] = await Promise.all([
      this.find({ ...options, limit, offset }),
      this.count(options.where, options.tx, options.withDeleted)
    ]);
    return { data, meta: { total, page, limit, totalPages: Math.ceil(total / limit) } };
  }

  // --- MÉTODOS PÚBLICOS DE ESCRITA ---
  async create(data: Partial<T>, tx?: PoolClient): Promise<T> {
    const processedData = await this.beforeInsert(data);
    const cleanData = Object.fromEntries(Object.entries(processedData).filter(([, v]) => v !== undefined));
    const keys = Object.keys(cleanData);
    if (keys.length === 0) throw new Error(`Dados inválidos para insert em ${this.tableName}`);

    const values = Object.values(cleanData);
    const placeholders = keys.map((_, i) => `$${i + 1}`).join(', ');
    const escapedColumns = keys.map(k => this.escapeId(k)).join(', ');
    
    const query = `INSERT INTO ${this.escapeId(this.tableName)} (${escapedColumns}) VALUES (${placeholders}) RETURNING *`;
    this.logQuery(query, values);
    
    const client = this.getClient(tx);
    const { rows } = await client.query(query, values);
    return rows[0];
  }

  async upsert(data: Partial<T>, conflictTarget: keyof T | Array<keyof T>, tx?: PoolClient): Promise<T> {
    const processedData = await this.beforeInsert(data);
    const cleanData = Object.fromEntries(Object.entries(processedData).filter(([, v]) => v !== undefined));
    const keys = Object.keys(cleanData);
    if (keys.length === 0) throw new Error('Dados inválidos para upsert');

    const values = Object.values(cleanData);
    const placeholders = keys.map((_, i) => `$${i + 1}`).join(', ');
    const escapedColumns = keys.map(k => this.escapeId(k)).join(', ');
    
    const targets = Array.isArray(conflictTarget) 
      ? conflictTarget.map(t => this.escapeId(t)).join(', ') 
      : this.escapeId(conflictTarget);

    let updateSets: string[] = keys
      .filter((k) => k !== this.primaryKey && k !== 'created_at' && !(this.useSoftDeletes && k === 'deleted_at'))
      .map((k) => `${this.escapeId(k)} = EXCLUDED.${this.escapeId(k)}`);

    if (this.useTimestamps) updateSets.push(`"updated_at" = NOW()`);
    if (updateSets.length === 0) updateSets.push(`${this.escapeId(this.primaryKey)} = EXCLUDED.${this.escapeId(this.primaryKey)}`);

    const query = `
      INSERT INTO ${this.escapeId(this.tableName)} (${escapedColumns})
      VALUES (${placeholders})
      ON CONFLICT (${targets})
      DO UPDATE SET ${updateSets.join(', ')}
      RETURNING *
    `;
    this.logQuery(query, values);
    const client = this.getClient(tx);
    const { rows } = await client.query(query, values);
    return rows[0];
  }

  async createMany(items: Partial<T>[], tx?: PoolClient): Promise<T[]> {
    if (items.length === 0) return [];
    const processedItems = await Promise.all(items.map(item => this.beforeInsert(item)));

    const columns: string[] = [];
    const seen = new Set<string>();
    for (const item of processedItems) {
      for (const [k, v] of Object.entries(item)) {
        if (v !== undefined && !seen.has(k)) {
          seen.add(k);
          columns.push(k);
        }
      }
    }
    if (columns.length === 0) return [];

    const allValues: any[] = [];
    const rowPlaceholders: string[] = [];
    let paramIndex = 1;

    for (const item of processedItems) {
      const row: string[] = [];
      for (const col of columns) {
        allValues.push((item as any)[col] ?? null);
        row.push(`$${paramIndex++}`);
      }
      rowPlaceholders.push(`(${row.join(', ')})`);
    }

    const escapedColumns = columns.map(c => this.escapeId(c)).join(', ');
    const query = `INSERT INTO ${this.escapeId(this.tableName)} (${escapedColumns}) VALUES ${rowPlaceholders.join(', ')} RETURNING *`;
    this.logQuery(query, allValues);
    const client = this.getClient(tx);
    const { rows } = await client.query(query, allValues);
    return rows;
  }

  async update(id: number | string, data: Partial<T>, tx?: PoolClient): Promise<T | undefined> {
    const processedData = await this.beforeUpdate(data);
    const cleanData = Object.fromEntries(Object.entries(processedData).filter(([, v]) => v !== undefined));
    const keys = Object.keys(cleanData);
    if (keys.length === 0) return this.findById(id, { tx });

    const values = Object.values(cleanData);
    const setClause = keys.map((key, i) => `${this.escapeId(key)} = $${i + 1}`).join(', ');
    
    const query = `UPDATE ${this.escapeId(this.tableName)} SET ${setClause} WHERE ${this.escapeId(this.primaryKey)} = $${keys.length + 1} RETURNING *`;
    const finalParams = [...values, id];
    
    this.logQuery(query, finalParams);
    const client = this.getClient(tx);
    const { rows } = await client.query(query, finalParams);
    return rows[0];
  }

  async updateMany(where: FindOptions<T>['where'], data: Partial<T>, tx?: PoolClient): Promise<number> {
    const processedData = await this.beforeUpdate(data);
    const cleanData = Object.fromEntries(Object.entries(processedData).filter(([, v]) => v !== undefined));
    const keys = Object.keys(cleanData);
    if (keys.length === 0) return 0;

    const setValues = Object.values(cleanData);
    const setClause = keys.map((key, i) => `${this.escapeId(key)} = $${i + 1}`).join(', ');

    const { whereClause, params } = this.buildWhere(where ?? {}, false, setValues);
    if (!whereClause) throw new Error('updateMany exige um "where" não vazio.');

    const query = `UPDATE ${this.escapeId(this.tableName)} SET ${setClause} ${whereClause}`;
    this.logQuery(query, params);
    
    const client = this.getClient(tx);
    const { rowCount } = await client.query(query, params);
    return rowCount ?? 0;
  }

  async delete(id: number | string, tx?: PoolClient): Promise<boolean> {
    const client = this.getClient(tx);
    if (this.useSoftDeletes) {
      const query = `UPDATE ${this.escapeId(this.tableName)} SET "deleted_at" = NOW() WHERE ${this.escapeId(this.primaryKey)} = $1 RETURNING ${this.escapeId(this.primaryKey)}`;
      this.logQuery(query, [id]);
      const { rowCount } = await client.query(query, [id]);
      return (rowCount ?? 0) > 0;
    }

    const query = `DELETE FROM ${this.escapeId(this.tableName)} WHERE ${this.escapeId(this.primaryKey)} = $1 RETURNING ${this.escapeId(this.primaryKey)}`;
    this.logQuery(query, [id]);
    const { rowCount } = await client.query(query, [id]);
    return (rowCount ?? 0) > 0;
  }

  async deleteMany(where: FindOptions<T>['where'], tx?: PoolClient): Promise<number> {
    const { whereClause, params } = this.buildWhere(where ?? {});
    if (!whereClause) throw new Error('deleteMany exige um "where" não vazio.');

    const client = this.getClient(tx);
    const query = this.useSoftDeletes
      ? `UPDATE ${this.escapeId(this.tableName)} SET "deleted_at" = NOW() ${whereClause}`
      : `DELETE FROM ${this.escapeId(this.tableName)} ${whereClause}`;

    this.logQuery(query, params);
    const { rowCount } = await client.query(query, params);
    return rowCount ?? 0;
  }

  async restore(id: number | string, tx?: PoolClient): Promise<boolean> {
    if (!this.useSoftDeletes) throw new Error('Soft Deletes não ativados.');
    const query = `UPDATE ${this.escapeId(this.tableName)} SET "deleted_at" = NULL WHERE ${this.escapeId(this.primaryKey)} = $1`;
    this.logQuery(query, [id]);
    const client = this.getClient(tx);
    const { rowCount } = await client.query(query, [id]);
    return (rowCount ?? 0) > 0;
  }

  // --- ESCAPE HATCH (SAÍDA DE EMERGÊNCIA) ---
  /** Quando você precisar rodar um SQL maluco que o framework não cobre */
  async queryRaw<R = any>(sql: string, params: any[] = [], tx?: PoolClient): Promise<R[]> {
    this.logQuery(sql, params);
    const client = this.getClient(tx);
    const { rows } = await client.query(sql, params);
    return rows;
  }

  // --- UNIT OF WORK ---
  async transaction<R>(callback: (tx: PoolClient) => Promise<R>): Promise<R> {
    const client = await pool.connect();
    try {
      await client.query('BEGIN');
      const result = await callback(client);
      await client.query('COMMIT');
      return result;
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }
}
```
