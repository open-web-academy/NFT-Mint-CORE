# dApp de Colección de NFTs en Core

Las aplicaciones descentralizadas (dApps) utilizan una blockchain o contratos inteligentes en cadena para almacenar y referenciar datos, en lugar de depender de bases de datos centralizadas tradicionales. Una estructura común y simple de una dApp generalmente consiste en un front-end en React.js o Vue.js que utiliza Web3.js o Ethers.js para interactuar con contratos inteligentes desplegados en una blockchain compatible con EVM.

## ¿Qué puedes hacer en este tutorial?

En este tutorial, aprenderás a crear una dApp para mintear NFTs en la red CORE.

* `Mintear NFT`: Aprende a mintear nuevos NFTs. Esto incluye la creación de tokens únicos, la asignación de metadatos y la interacción con contratos inteligentes para registrar NFTs en la blockchain de CORE.
* `Obtener NFTs`: Descubre cómo obtener y ver NFTs minteados Los usuarios podrán ver los detalles de sus NFTs, incluidos metadatos y propiedades únicas de cada token, todo esto directamente desde la dApp.

## Requisitos de Software

* [Git](https://git-scm.com/) v2.44.0
* [Node.js](https://nodejs.org/en) v20.11.1
* [npm](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm) v10.2.4
* [Hardhat](https://hardhat.org/hardhat-runner/docs/getting-started#installation) v10.2.4
* [MetaMask Web Wallet Extension](https://metamask.io/download/)

## Configuración del entorno de desarrollo

1. Descarga este repositorio

2. Instala las dependencias en la ruta /contract.
   
```bash
npm install
```

3. Instala y configura la Extensión MetaMask para Chrome para usar con Core Testnet. Consulta [aquí](https://docs.coredao.org/docs/Dev-Guide/core-testnet-wallet-config) para una guía detallada.

4. Crea un archivo secret.json en la carpeta /contract y almacena la clave privada de tu billetera MetaMask en él. Consulta [aquí](https://metamask.zendesk.com/hc/en-us/articles/360015290032-How-to-reveal-your-Secret-Recovery-Phrase) para detalles sobre cómo obtener la clave privada de la cuenta MetaMask. Ejemplo:

```json
{"PrivateKey":"ef1150b212a53b053a3dee265cb26cd010065b9340b4ac6cf5d895a7cf39c923"}
```

:::caution
No olvides añadir este archivo al archivo .gitignore en la carpeta raíz de tu proyecto para evitar que tu clave privada/frases secretas se suban accidentalmente a un repositorio público. ¡Asegúrate de mantener este archivo en un lugar absolutamente seguro!
:::

5. Copia lo siguiente en tu archivo hardhat.config.js en /contract

```js
/**
 * @type import('hardhat/config').HardhatUserConfig
 */


require('@nomiclabs/hardhat-ethers');
require("@nomiclabs/hardhat-waffle");


const { PrivateKey } = require('./secret.json');


module.exports = {
   defaultNetwork: 'testnet',


   networks: {
      hardhat: {
      },
      testnet: {
         url: 'https://rpc.test.btcs.network',
         accounts: [PrivateKey],
         chainId: 1115,
      }
   },
   solidity: {
      compilers: [
        {
           version: '0.8.24',
           settings: {
            evmVersion: 'paris',
            optimizer: {
                 enabled: true,
                 runs: 200,
              },
           },
        },
      ],
   },
   paths: {
      sources: './contracts',
      cache: './cache',
      artifacts: './artifacts',
   },
   mocha: {
      timeout: 20000,
   },
};
```

## Escritura del Contrato Inteligente

1. Dentro de la carpeta /contract/contracts se encuentra el archivo `NFTCollection.sol` que contendrá el código del contrato inteligente a utilizar en este tutorial.

```javascript
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/utils/Base64.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Counters.sol";

contract NFTCollection is ERC721URIStorage, Ownable {
    using Counters for Counters.Counter;
    Counters.Counter private _tokenIds;

    struct TokenURI {
        string tokenURI;
        string image;
    }

    struct NFTItem {
        string name;
        string description;
        string imageURI;
    }

    mapping(uint256 => NFTItem) private _tokenDetails;

    constructor() ERC721("NFTCollection", "NFTC") Ownable() {}

    function mintNFT(
        string memory name,
        string memory description,
        string memory imageURI
    ) public onlyOwner returns (uint256) {
        require(bytes(name).length > 0, "Name is required");
        require(bytes(description).length > 0, "Description is required");
        require(bytes(imageURI).length > 0, "Image URI is required");

        uint256 newItemId = _tokenIds.current();
        _mint(msg.sender, newItemId);

        TokenURI memory tokenURI = generateTokenURI(
            name,
            description,
            imageURI
        );

        _tokenDetails[newItemId] = NFTItem({
            name: name,
            description: description,
            imageURI: imageURI
        });

        _setTokenURI(newItemId, tokenURI.tokenURI);

        _tokenIds.increment();
        return newItemId;
    }

    function getTokenDetails(uint256 tokenId)
        public
        view
        returns (NFTItem memory)
    {
        require(_hasToken(tokenId), "Token does not exist");
        return _tokenDetails[tokenId];
    }

    function getNFTsByPage(uint256 page, uint256 pageSize)
        public
        view
        returns (NFTItem[] memory)
    {
        require(page > 0, "Page number should be greater than 0");
        uint256 startIndex = (page - 1) * pageSize;
        uint256 endIndex = startIndex + pageSize;
        uint256 totalItems = _tokenIds.current();

        if (endIndex > totalItems) {
            endIndex = totalItems;
        }

        require(startIndex < totalItems, "Page number out of range");

        NFTItem[] memory items = new NFTItem[](endIndex - startIndex);
        for (uint256 i = startIndex; i < endIndex; i++) {
            items[i - startIndex] = _tokenDetails[i];
        }

        return items;
    }

    function _hasToken(uint256 tokenId) internal view returns (bool) {
        try this.ownerOf(tokenId) returns (address) {
            return true;
        } catch {
            return false;
        }
    }

    function generateTokenURI(
        string memory name,
        string memory description,
        string memory image
    ) internal pure returns (TokenURI memory) {
        string memory json = string(
            abi.encodePacked(
                '{"name": "',
                name,
                '", "description": "',
                description,
                '", "image": "',
                image,
                '"}'
            )
        );

        string memory base64Json = Base64.encode(bytes(json));

        string memory tokenURI = string(
            abi.encodePacked("data:application/json;base64,", base64Json)
        );

        TokenURI memory tokenURIStruct = TokenURI(tokenURI, image);

        return tokenURIStruct;
    }
}
```
### Explicación

El contrato inteligente `NFTCollection` está diseñado para gestionar una colección de tokens no fungibles (NFTs) en la red de CORE. Utiliza las bibliotecas de OpenZeppelin para facilitar las funcionalidades estándar de ERC721 y la gestión del URI del token.

1. **Importaciones**:
   - `Base64`: Para codificar los datos JSON del URI del token en base64.
   - `ERC721URIStorage`: Una extensión de ERC721 que permite almacenar un URI para cada token.
   - `Ownable`: Proporciona control de acceso para funciones que solo el propietario puede ejecutar.
   - `Counters`: Se utiliza para generar identificadores únicos para los tokens.
  
2. **Estructuras de Datos**:
   - `TokenURI`: Almacena el URI codificado en base64 y la imagen asociada del token.
   - `NFTItem`: Contiene detalles sobre cada NFT, incluidos nombre, descripción y URI de la imagen.
  
3. **Variables de Estado**:
   - `_tokenIds`: Contador para gestionar la numeración de los tokens.
   - `_tokenDetails`: Mapeo que asocia un ID de token con sus detalles correspondientes.

4. **Funciones Principales**:
   - `mintNFT(string memory name, string memory description, string memory imageURI)`:  Permite al propietario del contrato mintear un nuevo NFT con un nombre, descripción y URI de la imagen. Devuelve el ID del token recién acuñado.
   - `getTokenDetails(uint256 tokenId)`: Permite consultar los detalles de un token específico dado su ID.
   - `getNFTsByPage(uint256 page, uint256 pageSize)`: Devuelve una lista paginada de NFTs. Los parámetros page y pageSize permiten especificar qué página de resultados recuperar y el número de elementos por página, respectivamente.


## Compilación del Contrato Inteligente

1. Para compilar el contrato inteligente `NFTCollectio`n definido en `NFTCollection.sol`, desde el directorio /contract ejecuta el siguiente comando. (Cada vez que se realice un cambio en el código del contrato, debemos recompilarlo).

```bash
npx hardhat compile
```

## Despliegue e Interacción con el Contrato Inteligente

1. Antes de desplegar tu contrato inteligente en la cadena Core, es aconsejable ejecutar una serie de pruebas para asegurarse de que el contrato funciona como se desea. Consulta la guía detallada [aquí](https://docs.coredao.org/docs/Dev-Guide/hardhat#contract-testing) para más detalles.

2. Crea una carpeta `scripts` en el directorio /contract de tu proyecto. Dentro de esta carpeta, crea un archivo `deploy.js`; pega el siguiente script en él.

```javascript
async function main() {
  const [deployer] = await ethers.getSigners();

  console.log("Deploy contract with the account:", deployer.address);

  const NFTCollection = await ethers.getContractFactory("NFTCollection");

  const nftCollection = await NFTCollection.deploy();

  console.log("Contract Address:", nftCollection.address);
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });

```
3. Asegúrate de que tu billetera MetaMask tenga tokens de prueba tCORE para la Red de Pruebas Core. Consulta [aquí](https://docs.coredao.org/docs/Dev-Guide/core-faucet) para detalles sobre cómo obtener tokens tCORE.

4. Ejecuta el siguiente comando desde el directorio raíz de tu proyecto para desplegar tu contrato inteligente en la cadena Core.

```bash
npx hardhat run scripts/deploy.js
```

## Configuración del Frontend

1. En la carpeta raíz, instala todas las dependencias.

```bash
npm install
```

2. En la ruta src/contractABI debemos copiar el ABI de nuestro contrato inteligente en caso de hacer modificaciones, esta información se obtendrá de contract/artifacts/contracts/NFTCollection.json.

3. Una vez que el contrato inteligente esté desplegado, es necesario copiar la dirección y reemplazarla en cada uno de los componentes donde realizamos llamadas al contrato, en este caso en src/components/Mint.tsx y src/components/Get.tsx.

4. Para probar si todo está funcionando correctamente, ejecuta la aplicación utilizando el siguiente comando. Esto servirá la aplicación con la función de recarga en caliente en [http://localhost:5173](http://localhost:5173/)

```bash
npm run dev
```

## Mintear NFT

 1. Para acuñar un nuevo NFT, solo tendrás que ingresar 3 campos: el nombre, la descripción y la URL de la imagen.
 2. Una vez hecho esto, haz clic en el botón “Mint” y acepta la transacción en MetaMask.

<img src="https://raw.githubusercontent.com/open-web-academy/NFT-Tutorial-CORE/master/src/public/mint.gif" width="50%">

## Obtener NFT's

1. Para obtener los NFTs minteados, solo debes ir a la opción de “Get NFTs”.

<img src="https://raw.githubusercontent.com/open-web-academy/NFT-Tutorial-CORE/master/src/public/get.gif" width="50%">
