# ansible-load-secrets

role tries to load any kind of secrets directly into variables you specify in the `vars_stored`.

Supports loading variable values from secrets stored in local __filesystem__ `(secret_store: 'fs')` or from the instance of HashiCorp __Vault__ `(secret_store: 'vault')`.

If your secret does not exist at the store, the variable WILL NOT BE SET (remains in original state - `undefined` or whatever you set it to previously).

## Specify the source

Set `als_secret_store` either to `vault` or `fs`. Loading values from both sources is intentionally not supported.

For `vault` source, specify mount point with `als_vault_mount` and path with `als_vault_path` variables.

For `fs` source, specify `als_fs_path`. If not specified, `'/tmp'` will be used as default.

## Sample plays

### Hashicorp vault source

Don't forget to specify `VAULT_ADDR` and `VAULT_TOKEN` env vars as you are used to when using Hashicorp Vault.

If you ever run into:
```
Exception: HTTPSConnectionPool(host='vault.example.com', port=443): 
Max retries exceeded with url: /v1/sys/seal-status (Caused by SSLError(
SSLError(\"bad handshake: Error([('SSL routines', 'tls_process_server_certificate', 'certificate verify failed')],)\",),))"
```
then define proper certification authority for Python Requests with `export REQUESTS_CA_BUNDLE=/etc/ssl/ca-certificates.crt`.

```
- hosts: all
  vars:
    - secret_store: 'vault'
    - als_vault_mount: 'secrets'
    # full path will be secrets/someproject/prod
    - als_vault_path: 'someproject/prod'
    - group: 'somehosts'
    - vars_stored:
      - var: 'secret1'
        key: 'password'
      - var: 'secret2'
        key: 'someotherkey'
      - var: 'secret3'
        key: 'someotherkey'
        token: '1234-5678-9012-3456'
  roles:
    - ansible-load-secrets
```

After this play, `{{ secret1 }}` will be picked from `{{ als_vault_mount }}/{{ als_vault_path }}/{{ group }}/secret1:password` and `{{ secret2 }}` will be picked from `{{ als_vault_mount }}/{{ als_vault_path }}/{{ group }}/secret2:someotherkey`.

You can specify an empty `group`, but if you have read the previous paragraph you surely understand this well.

`secret3` is loaded with a specific token (good for one-time usage).

### Filesystem source

```
- hosts: somehosts
  vars:
    - secret_store: 'fs'
    - group: somehosts
    - vars_stored:
      - var: 'secret1'
      - var: 'secret2'
  roles:
    - ansible-load-secrets
```

After this play, `{{ secret1 }}` will be loaded from `{{ als_fs_path }}/somehosts/secret1`, and `{{ secret2 }}` from `{{ als_fs_path }}/somehosts/secret2` respectively.

You can use empty `group`, useful for loading vars for all hosts:

```
- hosts: all
  vars:
    - secret_store: 'fs'
    - vars_stored:
      - var: 'secret1'
      - var: 'secret2'
  roles:
    - ansible-load-secrets
```

After this play, `{{ secret1 }}` will be loaded from `{{ als_fs_path }}/secret1`, and `{{ secret2 }}` from `{{ als_fs_path }}/secret2` respectively.

### Password generation

Role is able to auto-generate pseudo-random `{{ password_length }}`-chars long (default 10) passwords. To do so, specify `password: yes` in your item:

```
    - vars_stored:
      - var: 'topsecret'
        key: 'password'
        password: true
```

After this, if `topsecret` was not loaded from vault, you will receive 10-char random string.

You can specify item-specific password length as well:

```
    - vars_stored:
      - var: 'topsecret'
        key: 'password'
        password: true
        length: 42
```

It can generate random numbers a.k.a. PINs as well:

```
    - vars_stored:
      - var: 'pin'
        key: 'pin'
        pin: yes
```

Result: `pin` would be a number between 1000-9999

or

```
    - vars_stored:
      - var: 'pin'
        key: 'pin'
        pin: yes
        range_min: 1234
        range_max: 5678
```

would generate number between `1234` and `5677`.

Note: Ansible RNG will generate a number between `range_min` **inclusive** and `range_max` **exclusive**.

```
    - vars_stored:
      - var: 'pin'
        password: true
        chars: 'asdfgh'
```

will generate password consisting of chars from `asdfgh` set only (see `chars` in https://docs.ansible.com/ansible/2.5/plugins/lookup/password.html)

### Host-specific variable

```
    - vars_stored:
      - var: 'topsecret'
        key: 'password'
        host: 'myhost.mydomain.com'
```

Will load `topsecret` variable to `myhost.mydomain.com` on `myhost.mydomain.com` ONLY. Overrides `group` (if you specify both, only `host` is accepted).

Loading from special path: `{{ als_vault_mount }}/{{ als_vault_path }}/hostsecrets/{{ host }}/{{ var }}`

### Load secret from explicitly specified path

Sometimes you might need to load the secret from a path you define yourself. You can specify `path` parameter per item. In that case, `vault_mount` and `vault_path` are ignored. You need to specify full `path`, including the starting slash and mount point.

```
  vars:
    - vars_stored:
      - var: "arbitrary"
        key: "whatever"
        password: false
        path: '/secret/absolutely/random/path'
```

## License

GPL

## Author Information

Michal Medvecky  
Diogenes Santos de Jesus  