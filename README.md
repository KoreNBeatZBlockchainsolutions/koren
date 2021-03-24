# Token tutorial

This example dapp builds off learnings from our first tutorial, [Hello World](https://studio.ethereum.org/). We recommend you start there.

The goal of this tutorial is to teach you how to:

-   Write and deploy a smart contract that issues [tokens](https://docs.openzeppelin.com/contracts/2.x/tokens)
-   Build a web application that interacts with your smart contract
-   Understand the basics of [Ethereum accounts](https://ethereum.org/whitepaper/#ethereum-accounts) and how they represent both users and smart contracts on the blockchain
-   Transfer tokens between Ethereum accounts by initiating [transactions](https://ethereum.org/whitepaper/#messages-and-transactions) with your smart contract

If you'd like to learn more about how Ethereum works before diving in, we recommend you [start here](https://ethereum.org/learn/).

## Introduction to tokens

**What is a token?**

Ethereum tokens are simply digital assets that are built on top of the Ethereum blockchain using smart contracts. Tokens are used across many Ethereum projects.

**Why create a token?**

A token can serve many purposes,
such as a [stablecoin](https://www.investopedia.com/terms/s/stablecoin.asp),
or to track a point system (such as [Reddit's "community points"](https://www.reddit.com/community-points/)),
or even to represent shares of voting rights over a project. By representing things as tokens, we can allow smart contracts to interact with them, exchange them, create or destroy them.

You may have heard of [ERC20](https://docs.openzeppelin.com/contracts/2.x/erc20), a token standard established by the Ethereum community to promote interopability across applications.
In this example, you'll create a more simple token but we invite you to extend the contract code in order to satisfy the formal specification.

## The smart contract

At its core, a token contract is a mapping of addresses to balances, with some methods to add and subtract from those balances.

Let's find the [smart contract](https://ethereum.org/learn/#smart-contracts).

> Use the Explore panel on the left of the IDE to navigate to the _Files/contracts/Token.sol_ file.

Return here once you've read through the file.

Every smart contract runs at an address on the Ethereum blockchain. You must compile and deploy a smart contract to an address before it can run. When using Studio, your browser simulates the network, but there are several test networks and one main network for the Ethereum blockchain.

### 1. Compile

Solidity is a compiled language, and you need to convert the Solidity code into bytecode before the contract can run. We will automatically compile the code every time you save your contract (by clicking the floppy disk icon at the top of a file) or when performing a deployment.

### 2. Deploy

Now let's deploy the Token.sol contract. In the left panel of the IDE, you'll find the Deploy panel (the rocket icon). Deploy the contract by selecting the "Deploy" button within the Deploy panel.

You should now see successful output from the IDE's Console (lower right panel) and a web form in the IDE's Browser. This form allows you to interact with your deployed token contract.

## The web app (dapp)

As you learned in the Hello World example, dapps typically use a [JavaScript convenience libraries](https://ethereum.org/developers/#frontend-javascript-apis) that provide an API to make integrations with smart contract easier. In this project, you'll again be using [web3.js](https://web3js.readthedocs.io/en/v1.2.8/).

Let's take a look at our application logic.

> Use the Explore panel to navigate to the _Files/app/app.js_ file.

Return here once you've read through the file.

### Interact

Now that you have an understanding of the logic, let's use the app UI to interact with the contract!

**Find your Ethereum address**

At the top right of the IDE, find the Account dropdown menu (the "Default" account is pre-selected). Click to "Copy address" (the paper icon), then paste your address into the "Mint new tokens" address input field. Enter an amount and click "Mint".

Behind the scenes, submitting the form should trigger our JavaScript function, `createTokens`, which creates an Ethereum transaction to call the public function, `mint` on your smart contract and to update the contract's state.

**View you transactions**

Find the "Transactions" tab on the right panel of the IDE. Here you can inspect all transactions you've triggered on your deployed contract.

**Mint tokens**

Try using the Account dropdown menu to create a new Ethereum account, then transfer tokens to that account from your "Default" account (the account you deployed the contract on). You can verify if this worked using the "Check balance of address" form.

## Next Steps

Congratulations! You've made it through our second tutorial. You've taken another important step towards building on Ethereum.

Our final Ethereum Studio tutorial is our most complex example. We recommend you [build a "CryptoPizza NFT"](https://studio.ethereum.org/) next.

## Learn more

Want to learn more about Ethereum tokens?

-   [Read the ERC20 Token Standard](https://eips.ethereum.org/EIPS/eip-20)
-   [Learn how standards are established on Ethereum](https://ethereum.org/developers/#standards)
-   [See popular implementations of ERC20 interfaces, contracts and utilities](https://docs.openzeppelin.com/contracts/2.x/api/token/erc20)

Ready to move on from Ethereum Studio? Here's some other useful online resources to continue your dapp development journey:

-   [CryptoZombies](https://cryptozombies.io/): Learn Solidity building your own Zombie game
-   [Open Zeppelin Ethernaut](https://ethernaut.openzeppelin.com/): Complete levels by hacking smart contracts
-   [ChainShot](https://www.chainshot.com/): Solidity, Vyper and Web3.js coding tutorials
-   [Consensys Academy](https://consensys.net/academy/bootcamp/): Online Ethereum developer bootcamp
-   [Remix](https://remix.ethereum.org/): Web-based IDE for working on Solidity and Vyper smart contracts with in-line compile errors & code auto-complete

## Local development

Looking to set up a local Ethereum development environment? [Start here](https://ethereum.org/developers/#developer-tools).

// Copyright 2019 The go-ethereum Authors
// This file is part of the go-ethereum library.
//
// The go-ethereum library is free software: you can redistribute it and/or modify
// it under the terms of the GNU Lesser General Public License as published by
// the Free Software Foundation, either version 3 of the License, or
// (at your option) any later version.
//
// The go-ethereum library is distributed in the hope that it will be useful,
// but WITHOUT ANY WARRANTY; without even the implied warranty of
// MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
// GNU Lesser General Public License for more details.
//
// You should have received a copy of the GNU Lesser General Public License
// along with the go-ethereum library. If not, see <http://www.gnu.org/licenses/>.

package external

import (
	"fmt"
	"math/big"
	"sync"

	"github.com/ethereum/go-ethereum"
	"github.com/ethereum/go-ethereum/accounts"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/common/hexutil"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/event"
	"github.com/ethereum/go-ethereum/internal/ethapi"
	"github.com/ethereum/go-ethereum/log"
	"github.com/ethereum/go-ethereum/rpc"
	"github.com/ethereum/go-ethereum/signer/core"
)

type ExternalBackend struct {
	signers []accounts.Wallet
}

func (eb *ExternalBackend) Wallets() []accounts.Wallet {
	return eb.signers
}

func NewExternalBackend(endpoint string) (*ExternalBackend, error) {
	signer, err := NewExternalSigner(endpoint)
	if err != nil {
		return nil, err
	}
	return &ExternalBackend{
		signers: []accounts.Wallet{signer},
	}, nil
}

func (eb *ExternalBackend) Subscribe(sink chan<- accounts.WalletEvent) event.Subscription {
	return event.NewSubscription(func(quit <-chan struct{}) error {
		<-quit
		return nil
	})
}

// ExternalSigner provides an API to interact with an external signer (clef)
// It proxies request to the external signer while forwarding relevant
// request headers
type ExternalSigner struct {
	client   *rpc.Client
	endpoint string
	status   string
	cacheMu  sync.RWMutex
	cache    []accounts.Account
}

func NewExternalSigner(endpoint string) (*ExternalSigner, error) {
	client, err := rpc.Dial(endpoint)
	if err != nil {
		return nil, err
	}
	extsigner := &ExternalSigner{
		client:   client,
		endpoint: endpoint,
	}
	// Check if reachable
	version, err := extsigner.pingVersion()
	if err != nil {
		return nil, err
	}
	extsigner.status = fmt.Sprintf("ok [version=%v]", version)
	return extsigner, nil
}

func (api *ExternalSigner) URL() accounts.URL {
	return accounts.URL{
		Scheme: "extapi",
		Path:   api.endpoint,
	}
}

func (api *ExternalSigner) Status() (string, error) {
	return api.status, nil
}

func (api *ExternalSigner) Open(passphrase string) error {
	return fmt.Errorf("operation not supported on external signers")
}

func (api *ExternalSigner) Close() error {
	return fmt.Errorf("operation not supported on external signers")
}

func (api *ExternalSigner) Accounts() []accounts.Account {
	var accnts []accounts.Account
	res, err := api.listAccounts()
	if err != nil {
		log.Error("account listing failed", "error", err)
		return accnts
	}
	for _, addr := range res {
		accnts = append(accnts, accounts.Account{
			URL: accounts.URL{
				Scheme: "extapi",
				Path:   api.endpoint,
			},
			Address: addr,
		})
	}
	api.cacheMu.Lock()
	api.cache = accnts
	api.cacheMu.Unlock()
	return accnts
}

func (api *ExternalSigner) Contains(account accounts.Account) bool {
	api.cacheMu.RLock()
	defer api.cacheMu.RUnlock()
	if api.cache == nil {
		// If we haven't already fetched the accounts, it's time to do so now
		api.cacheMu.RUnlock()
		api.Accounts()
		api.cacheMu.RLock()
	}
	for _, a := range api.cache {
		if a.Address == account.Address && (account.URL == (accounts.URL{}) || account.URL == api.URL()) {
			return true
		}
	}
	return false
}

func (api *ExternalSigner) Derive(path accounts.DerivationPath, pin bool) (accounts.Account, error) {
	return accounts.Account{}, fmt.Errorf("operation not supported on external signers")
}

func (api *ExternalSigner) SelfDerive(bases []accounts.DerivationPath, chain ethereum.ChainStateReader) {
	log.Error("operation SelfDerive not supported on external signers")
}

func (api *ExternalSigner) signHash(account accounts.Account, hash []byte) ([]byte, error) {
	return []byte{}, fmt.Errorf("operation not supported on external signers")
}

// SignData signs keccak256(data). The mimetype parameter describes the type of data being signed
func (api *ExternalSigner) SignData(account accounts.Account, mimeType string, data []byte) ([]byte, error) {
	var res hexutil.Bytes
	var signAddress = common.NewMixedcaseAddress(account.Address)
	if err := api.client.Call(&res, "account_signData",
		mimeType,
		&signAddress, // Need to use the pointer here, because of how MarshalJSON is defined
		hexutil.Encode(data)); err != nil {
		return nil, err
	}
	// If V is on 27/28-form, convert to 0/1 for Clique
	if mimeType == accounts.MimetypeClique && (res[64] == 27 || res[64] == 28) {
		res[64] -= 27 // Transform V from 27/28 to 0/1 for Clique use
	}
	return res, nil
}

func (api *ExternalSigner) SignText(account accounts.Account, text []byte) ([]byte, error) {
	var signature hexutil.Bytes
	var signAddress = common.NewMixedcaseAddress(account.Address)
	if err := api.client.Call(&signature, "account_signData",
		accounts.MimetypeTextPlain,
		&signAddress, // Need to use the pointer here, because of how MarshalJSON is defined
		hexutil.Encode(text)); err != nil {
		return nil, err
	}
	if signature[64] == 27 || signature[64] == 28 {
		// If clef is used as a backend, it may already have transformed
		// the signature to ethereum-type signature.
		signature[64] -= 27 // Transform V from Ethereum-legacy to 0/1
	}
	return signature, nil
}

func (api *ExternalSigner) SignTx(account accounts.Account, tx *types.Transaction, chainID *big.Int) (*types.Transaction, error) {
	res := ethapi.SignTransactionResult{}
	data := hexutil.Bytes(tx.Data())
	var to *common.MixedcaseAddress
	if tx.To() != nil {
		t := common.NewMixedcaseAddress(*tx.To())
		to = &t
	}
	args := &core.SendTxArgs{
		Data:     &data,
		Nonce:    hexutil.Uint64(tx.Nonce()),
		Value:    hexutil.Big(*tx.Value()),
		Gas:      hexutil.Uint64(tx.Gas()),
		GasPrice: hexutil.Big(*tx.GasPrice()),
		To:       to,
		From:     common.NewMixedcaseAddress(account.Address),
	}
	if err := api.client.Call(&res, "account_signTransaction", args); err != nil {
		return nil, err
	}
	return res.Tx, nil
}

func (api *ExternalSigner) SignTextWithPassphrase(account accounts.Account, passphrase string, text []byte) ([]byte, error) {
	return []byte{}, fmt.Errorf("password-operations not supported on external signers")
}

func (api *ExternalSigner) SignTxWithPassphrase(account accounts.Account, passphrase string, tx *types.Transaction, chainID *big.Int) (*types.Transaction, error) {
	return nil, fmt.Errorf("password-operations not supported on external signers")
}
func (api *ExternalSigner) SignDataWithPassphrase(account accounts.Account, passphrase, mimeType string, data []byte) ([]byte, error) {
	return nil, fmt.Errorf("password-operations not supported on external signers")
}

func (api *ExternalSigner) listAccounts() ([]common.Address, error) {
	var res []common.Address
	if err := api.client.Call(&res, "account_list"); err != nil {
		return nil, err
	}
	return res, nil
}

func (api *ExternalSigner) pingVersion() (string, error) {
	var v string
	if err := api.client.Call(&v, "account_version"); err != nil {
		return "", err
	}
	return v, nil
}
