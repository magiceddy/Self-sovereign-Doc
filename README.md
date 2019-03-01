# Self-sovereign-Doc

Il caso d'uso permette ad un utente di crittografare tramite una coppia di chiavi (pubblica e privata) una rappresentazione della propria identità e salvarne l'hash su blockchain. Un utente potrà inoltre condividere tale identità con un Service Provider e potra delegarlo per condividerla verso terze parti.

Il framework si basa sul protocollo [Stow](https://stow-protocol.com/)

Di seguito verrà presentati gli step realizzativi.

1. In questo passaggio viene simulata la creazione della coppia di chiavi da parte di una JavaCard
```javascript
// creazione seed da una stringa mnemorica
const seed = bip39.mnemonicToSeed("possible cloud busy fashion report enact race congress pool enlist motion perfect");

//creazione wallet
const wallet = hdkey.fromMasterSeed(seed);

// per semplicità verrà utilizzato un solo wallet per tutti e tre gli utenti
const user =  wallet.derivePath("m/44'/60'/0'/0/99");
const provider1 =  wallet.derivePath("m/44'/60'/0'/0/100");
const provider2 = wallet.derivePath("m/44'/60'/0'/0/101");

// recupero le chiavi private dai singoli wallet
const userPrivKey = user.getWallet().getPrivateKey();
const provider1PrivKey = provider1.getWallet().getPrivateKey();
const provider1PrivKey = provider2.getWallet().getPrivateKey();

// recupero gli address dai singoli wallet
const userAddress = user.getWallet().getAddress();
const provider1Address = provider1.getWallet().getAddress();
const provider2Address = provider2.getWallet().getAddress();

// genero la coppia di chiavi per crittografare
const userKeyPairs = genKeyPairFromSecret(userPrivKey);
const provider1KeyPairs = genKeyPairFromSecret(provider1PrivKey);
const provider2KeyPairs = genKeyPairFromSecret(provider2PrivKey);
```

2. Registro l'utente, il Provider1 e Provider2 nello Smart Contract degli utenti
```javascript
await users.register({ from: userAddress });
await users.register({ from: provider1Address });
await users.register({ from: provider2Address });
```

3. Crypto l'identità tramite la chiave pubblica dell'utente e salvo l'identità su IPFS
```javascript
const identity = { name: 'Magic', surname: 'Eddy', age: '33' };
const encrypted = Stow.util.encrypt(userKeyPairs.publicKey, JSON.stringify(identity));

// salvo l'uri di riferimento su IPFS
const userDataUri = await ipfs.addJSONAsync(encrypted);
```

4. Salvo la rappresentazione dell'identità nello Smart Contract dei dati
```javascript

// creo il "manifesto" dei dati
const metadata = {
  dataFormat: "json",
  domain: "social media",
  storage: "IPFS",
  encryptionScheme: "x25519-xsalsa20-poly1305",
  encryptionPublicKey: [userKeyPairs.publicKey],
  stowjsVersion: "0.1.4",
  providerName: "",
  providerEthereumAddress: "",
  keywords: [ "socialmedia", "myIdentity",],
  creationDate: new Date()
};

// creo l'hash dei dati
const dataHash = stow.web3.utils.sha3(JSON.stringify(identity));

// salvo l'hash dei dati, il manifesto e l'uri per recuparare i dati su IPFS
await stow.addRecord(dataHash, metadata, userDataUri)
```

5. Leggo i riferimenti salvati su blockchain e recupero l'identità su IPFS
```javascript

// recupero i riferimenti tramite l'hash dell'identità
const userRecord = await stow.getRecord(dataHash);

// tramite l'uri recupero la rappresentazione criptata dei dati
const userDataEncryFromIPFS = await ipfs.catJSONAsync(userRecord.dataUri);

// decripto i dati tramite la chiave privata
const userDecryptIdentity = Stow.util.decrypt(userKeyPairs.privateKey, userDataEncryFromIPFS);
```

6. Condivido i dati al service privider criptandoli tramite la sua chiave pubblica e salvandoli su IPFS
```javascript
const privaderCryptoData = Stow.util.encrypt(provider1KeyPairs.publicKey, userDecryptIdentity);
const providerDataUri = await ipfs.addJSONAsync(privaderCryptoData);
```

7. Do accesso ai riferimenti salvati su blockchain al provider1
```javascript
await permissions.grantAccess(dataHash, providerAddress, providerDataUri);
```

8. Il provider recuper le informazioni su IPFS
```javascript
const providerDataEncryFromIPFS = await ipfs.catJSONAsync(ProviderDataUri);

// il provider1 decripta con la sua chiave privata
const providerDecrypt = Stow.util.decrypt(provider1KeyPairs.privateKey, providerDataEncryFromIPFS);
```

9. Delego il priveder uno a condividere i dati dell' utente tramite lo Smart Contract dei permessi
```javascript
await permissions.addDelegate(provider1Address);
```

10. Il priveder1 condivide le informazioni dell'utente con il privider2 firmandole con la chiave pubblica del provider2 e salva su IPFS
```javascript
const privader2CryptData = Stow.util.encrypt(provider2KeyPairs.publicKey, providerDecrypt);
const Provider2DataUri = await ipfs.addJSONAsync(privader2CryptData);
```

11. Il provider1 da l'accesso al provider2 per i dati riferiti all'utente tramite lo Smart Contract dei permessi
```javascript
await permissions.grantAccessbyDelegate(dataHash, provider2Address, userAddress, Provider2DataUri);
```

12. Il provider2 recupera le informazioni dell'utente su IPFS
```javascript
const provider2DataEncryFromIPFS = await ipfs.catJSONAsync(Provider2DataUri);
const provider2Decrypt = Stow.util.decrypt(provider2KeyPairs[2].privateKey, provider2DataEncryFromIPFS);
```

