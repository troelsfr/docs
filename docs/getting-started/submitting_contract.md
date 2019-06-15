# Developing smart contracts
We will discuss smart contracts examples in more details in one of the later tutorials, 
but before we dive into the examples, we will TODO.

## Developing contracts with vm-lang
TODO

We will develop a small test contract
```
@testCase
function main()
  createMessage("world");
  var greeting = persistentGreeting();

  if(greeting != "Hello, world!")
    panic("Greeting differed from expected message.");
  endif

  printLn(greeting);
endfunction

@init
function createMessage(name : String)
  var state = State< String >("greetings");
  state.set(name);
endfunction

@query
function persistentGreeting() : String
  var state = State< String >("greetings");
  return "Hello, " + state.get() + "!";
endfunction
```
You can test this contract by using the `vm-lang` executable. To test the script by running following from your 
build directory:
```
curl https://raw.githubusercontent.com/fetchai/etch-examples/master/01_submitting_contract/hello_world.etch --output hello_world.etch
./apps/vm-lang/vm-lang hello_world.etch
```
This produce an output similar to:
```
 F E ╱     vm-lang v0.4.1-rc3
   T C     Copyright 2018-2019 (c) Fetch AI Ltd.
     H

Hello, world!
```
We note that `vm-lang` executes `main` as the default function. When submitting the script to the ledger, we can leave `main` as it will not be accessible since it is not annotated as being invokable by the ledger code.

## Submitting the contract to the ledger
To submit the contract to the ledger, we develop a Python helper script that makes use of the Python API to interact with the ledger. We assume that the ledger will be running with 
```
import base64
import time
import hashlib
import json
import binascii
import msgpack

from fetchai.ledger.serialisation.objects.transaction_api import create_json_tx
from fetchai.ledger.api import ContractsApi, TransactionApi, submit_json_transaction
from fetchai.ledger.crypto import Identity

HOST = '127.0.0.1'
PORT = 8100

identity = Identity()
next_identity = Identity()

status_api = TransactionApi(HOST, PORT)
contract_api = ContractsApi(HOST, PORT)

create_tx = contract_api.create(identity, contract_source, init_resources = ["owner", identity.public_key])

print('CreateTX:', create_tx)

while True:
    status = status_api.status(create_tx)

    print(status)
    if status == "Executed":
        break

    time.sleep(1)

# re-calc the digest
hash_func = hashlib.sha256()
hash_func.update(contract_source.encode())
source_digest = base64.b64encode(hash_func.digest()).decode()

print('transfer N times')

for index in range(3):

    # create the tx
    tx = create_json_tx(
        contract_name=source_digest + '.' + identity.public_key + '.transfer',
        json_data=msgpack.packb([msgpack.ExtType(77, identity.public_key_bytes), msgpack.ExtType(77, next_identity.public_key_bytes), 1000 + index]),
        resources=['owner', identity.public_key, next_identity.public_key],
        raw_resources=[source_digest],
    )

    # sign the transaction contents
    tx.sign(identity.signing_key)

    wire_fmt = json.loads(tx.to_wire_format())
    print(wire_fmt)

    # # submit that transaction
    code = submit_json_transaction(HOST, PORT, wire_fmt)

    print(code)

    time.sleep(5)

print('Query for owner funds')

source_digest_hex = binascii.hexlify(base64.b64decode(source_digest)).decode()

url = 'http://{}:{}/api/contract/{}/{}/owner_funds'.format(HOST, PORT, source_digest_hex, identity.public_key_hex)

print(url)

r = status_api._session.post(url, json={})
print(r.status_code)
print(r.json())
```