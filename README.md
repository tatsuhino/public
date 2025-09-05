```


import { Client } from 'ldapts'

async function testLdapConnection() {
  const client = new Client({
    url: 'ldap://YOUR_SERVER:389',
    timeout: 30000,
    connectTimeout: 30000
  })

  try {
    // バインド
    await client.bind('YOUR_BIND_DN', 'YOUR_PASSWORD')
    console.log('✓ バインド成功')

    // RootDSEへの軽量クエリ
    const result = await client.search('', {
      filter: '(objectClass=*)',
      scope: 'base',
      attributes: ['namingContexts'],
      sizeLimit: 1,
      timeLimit: 1
    })
    console.log('✓ 検索成功:', result)

    // アンバインド
    await client.unbind()
    console.log('✓ アンバインド成功')
  } catch (error) {
    console.error('✗ エラー:', error)
  }
}

testLdapConnection()



```
