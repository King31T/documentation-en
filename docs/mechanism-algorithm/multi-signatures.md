# Account Permission Management

## Background

[Account permission management](https://github.com/tronprotocol/TIPs/blob/master/tip-16.md) functions allow for permission grading, and each permission can correspond to multiple private keys. This makes it possible to achieve multi-person joint control of accounts. This guide walks the user through TRON's account permission management implementation and design.

## Concept

There are three types of permission: owner、witness and active. Owner permission has the right to execute all the contracts. Witness permission is for SR. Active permission contains a set of contracts selected execution permissions.

### Protocol Definition

#### Account

```protobuf
message Account {
  // ...
  Permission owner_permission = 31;
  Permission witness_permission = 32;
  repeated Permission active_permission = 33;
}
```

Three attributes are added, owner_permission、witness_permission and active_permission. active_permission is a list, the length can not be bigger than 8.

#### ContractType

```protobuf
message Transaction {
  message Contract {
    enum ContractType {
      AccountCreateContract = 0;
      // ...
      AccountPermissionUpdateContract = 46;
    }
  }
}
```
The definition of ContractType can be found [here](https://github.com/tronprotocol/java-tron/blob/master/protocol/src/main/protos/core/Tron.proto).

AccountPermissionUpdateContract is a ContractType used to update the account permission.

#### AccountPermissionUpdateContract

```protobuf
message AccountPermissionUpdateContract {
  bytes owner_address = 1;
  Permission owner = 2;
  Permission witness = 3;
  repeated Permission actives = 4;
}
```

- `owner_address`: The address of the account whose permissions are to be modified
- `owner`: Owner permission
- `witness`: Witness permission (only used by witness)
- `actives`: Active permission

This will override the Original account permission. Therefore, if you only want to modify the owner permission, witness (if it is a SR account) and active permission also need to be set

#### Permission

```protobuf
message Permission {
  enum PermissionType {
    Owner = 0;
    Witness = 1;
    Active = 2;
  }
  PermissionType type = 1;
  int32 id = 2;
  string permission_name = 3;
  int64 threshold = 4;
  int32 parent_id = 5;
  bytes operations = 6;
  repeated Key keys = 7;
}
```

- `PermissionType`: Permission type
- `id`: Generated by system. Owner id=0, Witness id=1, Active id increases from 2. Specifying using which permission to execute a contract by setting id. For instance, using owner permission, set id=0
- `permission_name`: Permission name, 32 bytes length limit
- `threshold`: The threshold of the signature weight
- `parent_id`: Current 0
- `operations`: used by active permission, a hexadecimal coded sequence (little-endian byte order), 32 bytes (256 bits), and each bit represents the authority of a ContractType. The nth bit indicates the authority of the ContractType with ID n, its value 1 means that it has the authority to execute the ContractType, its value 0 means it doesn't have the authority.  
    To make it easier for users to read, start with the binary big-endian byte order to illustrate how to calculate the value of operations. The number of digits starts from 0, and corresponds to the ID of the ContractType from left to right. Convert a binary big-endian byte sequence to a hexadecimal little-endian byte sequence, that will be the value of operations. Below is an example of how to calculate the operations of active permission with operation TransferContract(ID=1) and operation VoteWitnessContract(ID=4) allowed. The mapping between ContractType and its ID can be seen in the above definition link of ContractType.

    | Operations Allowed  | Binary Code(big-endian) | Binary Code(little-endian) | Hex Code(little-endian) |
    | ------------- | ------------- | ------------- | ------------- |
    | TransferContract(1) & VoteWitnessContract(4)  | 01001000 00000000 00000000 ...  | 00010010 00000000 00000000 ... | 12 00 00 ... |

- `keys`: The accounts and weights that all own the permission, 5 keys at most.

#### Key

```protobuf
message Key {
  bytes address = 1;
  int64 weight = 2;
}
```

- `address`: The account address
- `weight`: The signature weight

#### Transaction

```protobuf
message Transaction {
  // ...
  int32 Permission_id = 5;
}
```

`Permission_id` is added. It is corresponding to `Permission.id`
1 is not allowed, because witness permission is only used to produce blocks, not for transaction signature.

### Owner Permission

Owner permission is the top permission of an account. It is used to control account ownership, adjust permission  structure. Owner Permission has the right to execute all the contracts.

Owner permission's features:

1. The account that has owner permission can change the owner permission
2. When owner permission is null, the default owner of the account owns the owner permission
3. When you create a new account, the address will be insert into owner permission automatically, default weight is 1, keys field only contains this address and also weight is 1.
4. If a permissionId is not specified when a contract is executed, using owner permission by default.

### Witness Permission

Super representatives can use this permission to manage block producing. Only SR(Super Representative) account has this permission.

Usage scenario example:
A super representative deploys a witness node on cloud server. In order to keep the account on the cloud server safe, you can only give the block producing permission to the account you put on cloud server. Because this account only owns  block producing permission, no TRX transfer permission, so even if the account on the cloud server is leaked, the TRX will not be lost.

Witness node configuration: when [start a fullnode as witness](https://tronprotocol.github.io/documentation-en/using_javatron/installing_javatron/#startup-a-fullnode-that-produces-blocks), `localwitness` in the config file is filled in with the private key of the witness account and `localWitnessAccountAddress` is commented on as below:
```
# config.conf
//localWitnessAccountAddress = 
localwitness = [
  xxx // private key of the witness account
]
```  

-  If witness permission is not modified, there is no need to change config file.  
-  If witness permission is modified, two modifications are required as follows:
    -  `localwitness` needs to be changed to the private key of the account authorized with witness permission
    - `localWitnessAccountAddress` should be explicitly set as the address of the witness account
   
    Below is an example of how to configure witness account [TCbxHgibJutCjVZUprvexKZZ4Rc6sJ4Xrk](https://nile.tronscan.org/#/address/TCbxHgibJutCjVZUprvexKZZ4Rc6sJ4Xrk) which authorize its witness permission to account TSwCH45gi2HvtqDYX3Ff39yHeu5moEqQDJ.
    ```
    #config.conf
    localWitnessAccountAddress = TCbxHgibJutCjVZUprvexKZZ4Rc6sJ4Xrk
    localwitness = [
      yyy // private key of TSwCH45gi2HvtqDYX3Ff39yHeu5moEqQDJ
    ]
    ```
    Notice: Only one private key can be added to `localwitness` when witness permission is modified.

### Active Permission

Active permission is composed of a set of contract execution permission, like creating an account, transfer function, etc.

Active permission's features:

1. the account owns owner permission can change active permission
2. the account owns the execution permission of AccountPermissionUpdateContract can also change active permission
3. 8 permissions at most
4. when a new account is created, an active permission will be created automatically, and the address will be inserted into it, default weight is 1, keys field only contains this address and weight is 1
5. permissionId increases from 2 automatically
6. When initiating a transaction using the Active permission, the permissionId must be explicitly set. The following is an example of a transaction where an account with the Active permission initiates resource delegation via trident, and the permissionId corresponding to this Active permission is 2:
```
package org.example;

import org.tron.trident.core.ApiWrapper;
import org.tron.trident.proto.Chain;
import org.tron.trident.proto.Contract.DelegateResourceContract;
import org.tron.trident.proto.Response;
import org.tron.trident.utils.Convert;

public class Main {

  public static void main(String[] args) {
    System.out.println("Hello world!");

    String agentPrivateKey = "your private key";
    String ownerAddress = "xxx";
    String receiverAddress = "yyy";
    long trxAmount = 10;
    int resourceType = 0;
    int permissionId = 2;

    try {
      ApiWrapper api = new ApiWrapper("grpc.trongrid.io:50051", "grpc.trongrid.io:50052", agentPrivateKey);
      long amountSun = Convert.toSun(String.valueOf(trxAmount), Convert.Unit.TRX).longValue();

      DelegateResourceContract contract = DelegateResourceContract.newBuilder()
          .setOwnerAddress(api.parseAddress(ownerAddress))
          .setReceiverAddress(api.parseAddress(receiverAddress))
          .setBalance(amountSun)
          .setResourceValue(resourceType)
          .setLock(false)
          .build();

      // create transaction extension
      Response.TransactionExtention txnExt = api.createTransactionExtention(
          contract,
          Chain.Transaction.Contract.ContractType.DelegateResourceContract
      );

      // get raw
      Chain.Transaction.raw.Builder rawBuilder = txnExt.getTransaction().getRawData().toBuilder();

      // set permission
      Chain.Transaction.Contract.Builder contractBuilder = rawBuilder.getContractBuilder(0)
          .setPermissionId(permissionId);

      // reset contract
      rawBuilder.setContract(0, contractBuilder.build());

      Chain.Transaction unsignedTxn = txnExt.getTransaction().toBuilder()
          .setRawData(rawBuilder.build())
          .build();

      // sign transaction
      Chain.Transaction signedTxn = api.signTransaction(unsignedTxn);

      Response.TransactionSignWeight transactionSignWeight = api.getTransactionSignWeight(signedTxn);
      Response.TransactionApprovedList transactionApprovedList = api.getTransactionApprovedList(signedTxn);

      System.out.println("transaction weight: " + transactionSignWeight);
      System.out.println("transaction approve list: " + transactionApprovedList);

    } catch (Exception e) {
      System.err.println("API init: " + e.getMessage());
      return;
    }
  }
}
```
### Fee

1. Using AccountPermissionUpdateContract costs 100TRX
2. If a transaction contains 2 or more than 2 signatures, it charges an extra 1 TRX besides the transaction fee
3. The fee can be modified by proposing

## API

### Change Permission

`AccountPermissionUpdateContract`, steps:

1. call `getaccount` to query the account, get the original permission
2. change permission
3. build transaction and sign
4. send transaction

Demo HTTP request:

```console
// POST to http://{{host}}:{{port}}/wallet/accountpermissionupdate

{
  "owner_address": "41ffa9466d5bf6bb6b7e4ab6ef2b1cb9f1f41f9700",
  "owner": {
    "type": 0,
    "id": 0,
    "permission_name": "owner",
    "threshold": 2,
    "keys": [{
        "address": "41F08012B4881C320EB40B80F1228731898824E09D",
        "weight": 1
      },
      {
        "address": "41DF309FEF25B311E7895562BD9E11AAB2A58816D2",
        "weight": 1
      },
      {
        "address": "41BB7322198D273E39B940A5A4C955CB7199A0CDEE",
        "weight": 1
      }
    ]
  },
  "witness": {
      "type": 1,
      "id": 1,
      "permission_name": "witness",
      "threshold": 1,
      "keys": [{
          "address": "41F08012B4881C320EB40B80F1228731898824E09D",
          "weight": 1
        }
      ]
    },
  "actives": [{
    "type": 2,
    "id": 2,
    "permission_name": "active0",
    "threshold": 3,
    "operations": "7fff1fc0037e0000000000000000000000000000000000000000000000000000",
    "keys": [{
        "address": "41F08012B4881C320EB40B80F1228731898824E09D",
        "weight": 1
      },
      {
        "address": "41DF309FEF25B311E7895562BD9E11AAB2A58816D2",
        "weight": 1
      },
      {
        "address": "41BB7322198D273E39B940A5A4C955CB7199A0CDEE",
        "weight": 1
      }
    ]
  }]
}
```

#### Calculate the Active Permission's Operations

```java
public static void main(String[] args) {

  //you need to specify the id of the contract you need to give permission to by referring to the definition of Transaction.ContractType in proto to get the id of the contract, below includes all the contract except AccountPermissionUpdateContract(id=46)

  Integer[] contractId = {0, 1, 2, 3, 4, 5, 6, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 30, 31,
      32, 33, 41, 42, 43, 44, 45};
  List<Integer> list = new ArrayList<>(Arrays.asList(contractId));
  byte[] operations = new byte[32];
  list.forEach(e -> {
    operations[e / 8] |= (1 << e % 8);
  });

  //7fff1fc0033e0000000000000000000000000000000000000000000000000000
  System.out.println(ByteArray.toHexString(operations));
}
```

### Contract Execution

(1). Create transaction, the same as none multi-signatures

(2). Specify `Permission_id`, default 0, represent owner permission, [demo](https://github.com/tronprotocol/wallet-cli/commit/ff9122f2236f1ce19cbb9ba9f0494c8923a3d10c#diff-a63fa7484f62fe1d8fb27276c991a4e3R211)

(3). User A sign the transaction, and then send it to user B

(4). User B sign the transaction gets from A, and then send it to user C

......

(n). The last users that signs the transaction broadcast it to the node

(n+1). The node will verify if the sum of the weight of all signatures is bigger than threshold, if true, the transaction is accepted, otherwise, is rejected

### Other APIs

Please refer to [HTTP API](../api/http.md) and [RPC API](../api/rpc.md) for more information.


1. query the addresses that already signed a transaction

    ```console
    > curl -X POST http://127.0.0.1:8090/wallet/getapprovedlist -d '{"transaction"}'
    ```

    ```protobuf
    rpc GetTransactionApprovedList(Transaction) returns (TransactionApprovedList) { }
    ```

2. query the signature weight of a transaction

    ```console
    > curl -X POST http://127.0.0.1:8090/wallet/getsignweight -d '{"transaction"}'
    ```

    ```protobuf
    rpc GetTransactionSignWeight (Transaction) returns (TransactionSignWeight) {}
    ```

