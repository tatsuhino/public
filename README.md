```
import { Client } from 'ldapts'
import type { LDAPConfig } from './types'

interface PooledConnection {
  client: Client
  inUse: boolean
  lastUsed: Date
  id: number
}

export class LDAPConnectionPool {
  private connections: PooledConnection[] = []
  private config: LDAPConfig
  private maxConnections: number
  private minConnections: number
  private connectionTimeout: number
  private idleTimeout: number
  private nextId = 1

  constructor(
    config: LDAPConfig,
    options: {
      maxConnections?: number
      minConnections?: number
      connectionTimeout?: number
      idleTimeout?: number
    } = {}
  ) {
    this.config = config
    this.maxConnections = options.maxConnections || 10
    this.minConnections = options.minConnections || 2
    this.connectionTimeout = options.connectionTimeout || 30000 // 30秒
    this.idleTimeout = options.idleTimeout || 300000 // 5分

    // 初期接続を作成
    this.initializePool()

    // アイドル接続のクリーンアップを定期的に実行
    setInterval(() => this.cleanupIdleConnections(), 60000) // 1分ごと
  }

  private async initializePool() {
    const promises = []
    for (let i = 0; i < this.minConnections; i++) {
      promises.push(this.createConnection())
    }
    await Promise.all(promises)
  }

  private async createConnection(): Promise<PooledConnection | null> {
    try {
      const client = new Client({
        url: this.config.url,
        timeout: this.connectionTimeout,
        connectTimeout: this.connectionTimeout
      })

      console.log('createConnection', this.config.url, this.config.bindDN)

      // bindDNとbindPasswordでバインド
      await client.bind(this.config.bindDN, this.config.bindPassword)

      const connection: PooledConnection = {
        client,
        inUse: false,
        lastUsed: new Date(),
        id: this.nextId++
      }

      this.connections.push(connection)
      return connection
    } catch (error) {
      console.error('Failed to create LDAP connection:', error)
      return null
    }
  }

  async acquire(options: { maxRetries?: number; retryDelay?: number } = {}): Promise<Client | null> {
    const { maxRetries = 19, retryDelay = 100 } = options

    for (let attempt = 0; attempt <= maxRetries; attempt++) {
      // 利用可能な接続を探す
      let connection = this.connections.find((conn) => !conn.inUse)

      if (!connection && this.connections.length < this.maxConnections) {
        // 新しい接続を作成
        const newConnection = await this.createConnection()
        if (newConnection) connection = newConnection
      }

      if (connection) {
        connection.inUse = true
        connection.lastUsed = new Date()

        // 接続の有効性を確認（RootDSEへの軽量なクエリでバインド状態を確認）
        try {
          await connection.client.search('', {
            filter: '(objectClass=*)',
            scope: 'base',
            attributes: ['namingContexts'],
            sizeLimit: 1,
            timeLimit: 1
          })
        } catch (error) {
          const errorMessage = error instanceof Error ? error.message : 'Unknown error'
          console.warn(`Connection ${connection.id} is invalid (${errorMessage}), reconnecting...`)
          // 既存の接続を確実にクリーンアップ
          try {
            await connection.client.unbind()
          } catch {}
          // 新しいクライアントを作成して再バインド
          connection.client = new Client({
            url: this.config.url,
            timeout: this.connectionTimeout,
            connectTimeout: this.connectionTimeout
          })

          try {
            await connection.client.bind(this.config.bindDN, this.config.bindPassword)
            console.log(`Connection ${connection.id} successfully reconnected`)
          } catch (bindError) {
            console.error(`Failed to rebind connection ${connection.id}:`, bindError)
            // バインドに失敗した場合は、この接続を削除して新しいものを作成
            const index = this.connections.indexOf(connection)
            if (index > -1) {
              this.connections.splice(index, 1)
            }
            return (
              (await this.createConnection()?.then((newConn) => {
                if (newConn) {
                  newConn.inUse = true
                  newConn.lastUsed = new Date()
                  return newConn.client
                }
                return null
              })) || null
            )
          }
        }

        return connection.client
      }

      // 利用可能な接続がない場合、リトライする
      if (attempt < maxRetries) {
        console.warn(
          `No available LDAP connections in pool, retrying in ${retryDelay}ms... (attempt ${attempt + 1}/${maxRetries + 1})`
        )
        await new Promise((resolve) => setTimeout(resolve, retryDelay))
      }
    }

    // すべてのリトライが失敗した場合
    console.error('Failed to acquire LDAP connection after all retries')
    return null
  }

  release(client: Client) {
    const connection = this.connections.find((conn) => conn.client === client)
    if (connection) {
      connection.inUse = false
      connection.lastUsed = new Date()
    }
  }

  private async cleanupIdleConnections() {
    const now = new Date()

    const idleConnections = this.connections.filter(
      (conn) =>
        !conn.inUse &&
        now.getTime() - conn.lastUsed.getTime() > this.idleTimeout &&
        this.connections.filter((c) => !c.inUse).length > this.minConnections
    )

    for (const connection of idleConnections) {
      try {
        await connection.client.unbind()
      } catch (error) {
        console.error(`Failed to unbind connection ${connection.id}:`, error)
      }
      const index = this.connections.indexOf(connection)
      if (index > -1) {
        this.connections.splice(index, 1)
        console.log(`Removed idle connection ${connection.id} after ${this.idleTimeout / 1000}s`)
      }
    }
  }

  async destroy() {
    // すべての接続を閉じる
    console.debug('destroy ldap connections length:', this.connections.length)
    const promises = this.connections.map(async (connection) => {
      try {
        await connection.client.unbind()
      } catch (error) {
        console.error(`Failed to unbind connection ${connection.id}:`, error)
      }
    })
    await Promise.all(promises)
    this.connections = []
  }

  getPoolStatus() {
    return {
      total: this.connections.length,
      inUse: this.connections.filter((c) => c.inUse).length,
      available: this.connections.filter((c) => !c.inUse).length
    }
  }
}

```
