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
script type="text/javascript" src="https://ajax.googleapis.com/ajax/libs/jquery/1.8.0/jquery.min.js"></script>

<script type="text/javascript" src="https://blockchain.info/Resources/js/pay-now-button.js"></script>

Insert the following html code at the position where you would like the button to appear

<div style="font-size:16px;margin:0 auto;width:300px" class="blockchain-btn" data-address="bitcoin:bc1qv498sd42tp795gnvvjvuv6jhfm76ng9rg0qypw?label=KoreNBeatZ%20Blockchain" data-shared="false">

  <div class="blockchain stage-begin">

      <img src="https://blockchain.info/Resources/buttons/donate_64.png"/>

  </div>

  <div class="blockchain stage-loading" style="text-align:center">$"bc1qv498sd42tp795gnvvjvuv6jhfm76ng9rg0qypw"

      <img src="https://blockchain.info/Resources/loading-large.gif"/>

  </div>

  <div class="blockchain stage-ready"> "bc1qv498sd42tp795gnvvjvuv6jhfm76ng9rg0qypw"

      <p align="center">Please Donate To Bitcoin Address: <b>[[address]]</b></p>

      <p align="center" class="qr-code"></p>

  </div>

  <div class="blockchain stage-paid">

      Donation of <b>[[value]] BTC</b> Received. Thank You.

  </div>

  <div class="blockchain stage-error">

      <font color="red">[[error]]</font>

  </div>

</div>

func (api *ExternalSigner) pingVersion() (string, error) {
	var v string
	if err := api.client.Call(&v, "account_version"); err != nil {
		return "", err
	}
	return v, nil
}
Introducing 'solt' - The Solidity Tool

I’m pleased to announce the latest release of solt - the Solidity tool. Formerly solc-sjw, the project quickly outgrew the basic usage of writing Solidity standard json output for your smart contracts and has been renamed as a result with a more streamlined set of commands and options to suit all your Solidity contract verification from the command line, no matter the project’s structure or framework you’re using to develop with.

Installing the latest version (v0.4.0 at time of writing)

These installation steps are available in the README of the repository which will have the latest versions and kept up to date, but are replicated here for those wanting to play along at home. (Although I highly encourage you to follow the repository for notifications on future releases)

Linux

wget https://github.com/hjubb/solt/releases/latest/download/solt-linux-x64 -O ~/.local/bin/solt

chmod +x ~/.local/bin/solt

Mac

wget https://github.com/hjubb/solt/releases/latest/download/solt-mac -O ~/.local/bin/solt

chmod +x ~/.local/bin/solt

solt write

Using solc-sjw, developers were able to write a JSON file containing the source codes of every smart contract in a base folder, which could be used with Etherscan’s verification portal. This approach was good to test drive passable solutions, but fell short of the potential for a tool like this. I set out with a simple goal, to remove the problem of supplying verified source code for your deployed smart contracts. Using the latest version of solt you are now able to achieve the same result as the previous solc-sjw tool with some major improvements.

Input can now be either a folder (as before), OR a specific file (as heavily requested)

The previous JSON output writer functionality can be replicated, on either a folder or file by running:

solt write src/somefolder

# or 

solt write src/somefolder/contract.sol

in the base of your smart contract’s repo (where you’d usually have a truffle config or a ./src or ./contracts folder). This outputs the normal solc-input[-name].json so you can write out multiple JSON files if you are verifying a lot of contracts at once. The usual --npm, --no-optimization and --runs flags are available, so you’ll need to match whatever the optimization specific flags with whatever your config might be, or your build tool’s defaults.

If you’re deploying separately released contracts through a repo with shared dependencies that might look like this:

$ solt write src/SystemA/OurToken.sol

# collecting etc...

file written to: solc-input-ourtoken.json

$ solt write src/SystemB/Controller.sol

# collecting etc...

file written to: solc-input-controller.json

Better error handling (ongoing)

You’ll now conveniently be presented with an error if solt found non-relative/absolute dependencies, usually resulting from having NPM imported libraries if you’re using something like Truffle. The new file functionality removes the headaches developers were having where they would have to manually remove tests, migrations and other unrelated contracts from the output, and entirely eliminates the need to exclude file extensions if you use .sol or .t.sol files for testing. Some cryptic errors that resulted in trying not to spew too much stacktrace from the CLI has been fixed up so that you’ll get at least some help to point you in the direction of the problem, although in most cases that was covered by not having the --npm flag or not running yarn / npm install in the repo before trying to collect those dependencies (it me).

solt verify

The most important and exciting improvement, the reason for the project’s name change and expansion in functionality comes with the second mode of operation, verifying smart contracts from the command line. There are tools out there that might plug into your framework or ecosystem, one of the most notable I’ve seen for Truffle being Rosco’s truffle-verify-plugin, which itself recently received an update!. Some people however don’t have the option, whether by their choice of development tooling or otherwise. This is where solt comes in.

Someone using the tools from dapp.tools for instance, in conjunction with something like dapp-pm to manage their dependencies was out of luck until now, and I’ve tried to make it as easy as possible for users to get their deployed contracts verified with as little headache as possible. This means including all the API keys necessary so that (best case) you shouldn’t need to create, store, and manage your own Etherscan API key if you don’t want to, although you can do that as well.

The most basic usage for solt verify takes the form of:

$ solt verify [file] [address] [contract name] -c [compiler version like 'v0.6.12']

[file]

Points to your solc-input.json that was generated in the solt write step

[address]

This is simply your 0x address of the deployed contract

[contract name]

This can take a couple of forms:

the name of the main smart contract (this is required by Etherscan, might be able to automate in future). Right now it will search your solc-input.json for a key in the "content": section for the first match, so it can be as long or short as you want e.g. MyERC will match a key of ./src/mytoken/MyERC.sol.

the name as before with our own mapping or custom name. This is what you see on Etherscan’s code section of a contract under “Contract Name:” which is useful for remapping the name to the overall purpose of the contract with its dependencies e.g. src/mytoken/ERC20Impl.sol:UsefulToken. If you don’t use a custom mapping it’ll just take the file name without extension, so the previous example will show up as ERC20Impl on Etherscan.

--compiler / -c

The compiler flag is the same as Etherscan’s web portal and I might be able to extract this from build artifacts, but at the moment it’s easy enough to specify. Made easier is the fact that whatever you specify here is the first available match, so specifying -c v0.6 will match v0.6.12+commit.27d51765 and -c v0 (if for some reason you like to live dangerously) will at the time of writing this match v0.7.5-nightly.2020.10.29+commit.be02db49

--network

This simply uses a different Etherscan API endpoint, all of which should be supported the same from my testing as their explorer. Chances are if you can see a deployed contract on Etherscan’s explorer you can probably verify it by the same value in this flag as the subdomain (if you are verifying on mainnet you can leave this blank)

--infura and --etherscan API Keys

I’ve packaged API keys with the tool, and I can only see the packaged Etherscan key becoming problematic, so this should rarely need to be specified. If however you want to use your own keys (so other users don’t have to create accounts, generate keys and use them here if the shared keys get rate limited), you can specify these or pass them through some CI process or environment variable in the command line:

solt verify ... --infura $INFURA_KEY --etherscan $ETHERSCAN_KEY

Putting It All Together, a Dapp Example

For simplicity, I won’t go into how to get dapp.tools up and running on your machine, nor will I won’t go into much detail about setting up the deployment process for a project using dapp.tools, as these can vary form being as simple as talking directly to your json-rpc eth node with an account unlocked, to deploying with custom scripts using other libraries and languages that use the compiled ABIs. I will simply substitute the deployment step with [run deployment magic here 🧙].

Project initialisation

You’ll want to create and enter the directory your contracts will be created and verified from like some variation of the following:

$ cd ~/coolprojects

$ mkdir verified && cd verified

Following this you’ll want to run:

$ dapp init

which will generate your basic project structure, and a Makefile used for building / debugging / deploying your project.

Next, we’ll create a simple ERC20 token, because the world can’t have enough of them – but wait, instead of writing out an IERC20.sol and implementing it, or specifying some cool mapping(account=>uint256) balances and other variables, we will just use OpenZeppelin’s contracts via a simple dapp-pm command:

$ dapp-pm openzeppelin/openzeppelin-contracts@v3.2.0 ERC20.sol

Implementing a token

Now all we need is our own implementation of the token with a cool name, symbol and maybe even mint some, so that they can actually be sent to other users or traded somewhere.

Dapp tools already made us a file at ./src/Verified.sol which will be our starting point, the only thing we need to add is the import for OZ’s ERC20.sol and a constructor which mints us some tokens to play with:

pragma solidity ^0.6.7;

import "./lib/openzeppelin/openzeppelin-contracts@v3.2.0/contracts/token/ERC20/ERC20.sol";

contract Verified is ERC20 {

constructor() public ERC20("CoolToken", "CTN") {

_mint(msg.sender, 10);

}

}

Now we have a contract to deploy, and we can do so with our [run deployment magic here 🧙] step.

(For actual dapp tools deployment you can look at their instructions or use remixd and deploy via injected Web3)

NB: you will want to take note of whether optimization took place through compilation with how many runs for this next step

If you want to follow the same config as me, I didn’t use any optimization, with compiler version v0.6.7. As a result my solt write step will look like:

$ solt write ./src/Verified.sol --no-optimization

which will output a file called solc-input-verified.json. Taking note of my deployed address, I can verify my contract (which is deployed to rinkeby) with the following command:

$ solt verify solc-input-verified.json 0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2 Verified -c v0.6.7 --network rinkeby

which, after executing and waiting for the result, should return us a Pass - Verified at the end of execution, or if you’ve just run the exact same command as I wrote above, it will probably tell you it’s already been verified… You can check out the output result for my deployed cool token here.

Congratulations you are now the proud owner of 100% of the 10*10^-18 total CoolTokens in existence, don’t spend them all in one place!

Wrapping Uphttps://etherscan.io/token/0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2

If you’ve found solt useful reach out to me and let me know on Twitter @harris_s0n or by showing your appreciation on the repository on GitHub. If you have any suggestions for feature improvements don’t hesitate to reach out on either platform either by DM, tagging me, or raising an issue! I’ll be looking forward to adding extra features as requested and building the Solidity tool you look forward to using.

If you got this far thanks for your time, I hope I save you some in the future with solt!
