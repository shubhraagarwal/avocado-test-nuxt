<template>
  <div>
    <p>EOA address : {{ this.eoaAddress }}</p>
    <p>Avocado address : {{ this.avocadoAddress }}</p>
    <button @click="getBalance(this.eoaAddress)">Get EOA balance</button>
    <button @click="getBalance(this.avocadoAddress)">
      Get Avocado balance
    </button>
    <button @click="transferAll">Transfer</button>
    <p>EOA Balance</p>
    <ul>
      <li v-for="(balance, index) in eoaBalance" :key="index">
        <p>{{ balance.chain }}</p>
        <ul>
          <li v-for="(token, index) in balance.data" :key="index">
            <p>{{ token.contract_name }}</p>
            <p>{{ token.contract_ticker_symbol }}</p>
            <p>{{ Number(token.balance) / 10 ** 18 }}</p>
          </li>
        </ul>
      </li>
    </ul>
    <p>Avocado Balance</p>
    <ul>
      <li v-for="(balance, index) in avocadoBalance" :key="index">
        <p>{{ balance.chain }}</p>
        <ul>
          <li v-for="(token, index) in balance.data" :key="index">
            <p>{{ token.contract_name }}</p>
            <p>{{ token.contract_ticker_symbol }}</p>
            <p>{{ Number(token.balance) / 10 ** 18 }}</p>
          </li>
        </ul>
      </li>
    </ul>
  </div>
</template>

<script>
import { ethers } from "ethers";
import { CovalentClient } from "@covalenthq/client-sdk";

export default {
  data() {
    return {
      eoaAddress: null,
      avocadoAddress: null,
      eoaBalance: [],
      avocadoBalance: [],
    };
  },
  async created() {
    this.eoaAddress = await this.getMetamask();
    await this.createAvocadoAddress(this.eoaAddress);
  },
  methods: {
    async getMetamask() {
      if (process.client && window.ethereum) {
        try {
          // Request account access
          await window.ethereum.enable();
          this.ethereum = window.ethereum;

          // After user grants account access, you can get the accounts.
          const accounts = await window.ethereum.request({
            method: "eth_accounts",
          });

          // accounts[0] is the address of the account currently connected to MetaMask
          return accounts[0];
        } catch (error) {
          console.error("User denied account access");
        }
      } else {
        console.error(
          "Non-Ethereum browser detected. You should consider trying MetaMask!"
        );
      }
    },
    async createAvocadoAddress(eoaAddress) {
      if (process.client && window.ethereum) {
        const forwarderABI = [
          {
            inputs: [
              {
                internalType: "address",
                name: "owner_",
                type: "address",
              },
              {
                internalType: "uint32",
                name: "index_",
                type: "uint32",
              },
            ],
            name: "computeAvocado",
            outputs: [
              {
                internalType: "address",
                name: "",
                type: "address",
              },
            ],
            stateMutability: "view",
            type: "function",
          },
        ];

        const provider = new ethers.JsonRpcProvider(
          "https://rpc.ankr.com/polygon"
        );
        const avoProvider = new ethers.JsonRpcProvider(
          "https://rpc.avocado.instadapp.io"
        );
        console.log(provider, "SDafsdf");

        const forwarderContractAddress =
          "0x46978CD477A496028A18c02F07ab7F35EDBa5A54";

        const contract = new ethers.Contract(
          forwarderContractAddress,
          forwarderABI,
          provider
        );

        this.avocadoAddress = await contract.computeAvocado(eoaAddress, 0);

        console.log(
          await avoProvider.send(
            "api_getSafes",
            [{ address: eoaAddress }],
            ":dasdas"
          )
        ); // All deployed safes
      }
    },
    async transferAll() {
      await this.switchNetwork("0x13881");
      await this.transferTokens(this.eoaBalance[1].data[0].balance);
      await this.switchNetwork("0xaa36a7");
      await this.transferTokens(this.eoaBalance[0].data[0].balance);
    },
    async switchNetwork(chainId) {
      const networks = {
        "0x13881": {
          chainName: "Mumbai Testnet",
          nativeCurrency: {
            name: "MATIC",
            symbol: "MATIC",
            decimals: 18,
          },
          rpcUrls: ["https://rpc-mumbai.maticvigil.com"],
          blockExplorerUrls: ["https://mumbai.polygonscan.com/"],
        },
        "0xaa36a7": {
          chainName: "Ethereum Sepolia",
          nativeCurrency: {
            name: "ETH",
            symbol: "ETH",
            decimals: 18,
          },
          rpcUrls: ["https://1rpc.io/sepolia"],
          blockExplorerUrls: ["https://sepolia.etherscan.com/"],
        },
      };

      const network = networks[chainId];

      try {
        await window.ethereum.request({
          method: "wallet_switchEthereumChain",
          params: [{ chainId: chainId }], // 0x4 is the chain ID for Rinkeby
        });
      } catch (switchError) {
        // This error code means the user needs to add the network.
        if (switchError.code === 4902) {
          try {
            await window.ethereum.request({
              method: "wallet_addEthereumChain",
              params: [
                {
                  chainId: chainId,
                  chainName: network.chainName,
                  nativeCurrency: {
                    name: network.nativeCurrency.name,
                    symbol: network.nativeCurrency.symbol,
                    decimals: 18,
                  },
                  rpcUrls: [network.rpcUrls[0]],
                  blockExplorerUrls: [network.blockExplorerUrls[0]],
                },
              ],
            });
          } catch (addError) {
            console.log(addError);
          }
        }
      }
    },
    async transferTokens(balance) {
      if (process.client && window.ethereum) {
        const provider = new ethers.BrowserProvider(window.ethereum);
        const signer = await provider.getSigner();
        const gasAmount = await provider.estimateGas({
          to: this.avocadoAddress,
          value: await provider.getBalance(this.eoaAddress),
        });
        // const gasPrice = (await provider.getFeeData()).gasPrice;

        // const gasFeeInWei = gasPrice * gasAmount;
        // console.log(gasFeeInWei, "gasFeeInWei", gasPrice, "gasPrice");
        const transaction = {
          to: this.avocadoAddress,
          value: `${0.1 * 10 ** 18}`,
        };
        const tx = await signer.sendTransaction(transaction);
        const receipt = await tx.wait();
        console.log("Reciept: ", receipt);
      }
    },
    async getBalance(address) {
      const client = new CovalentClient("cqt_rQMrFKW3YpHMk4X7VcgvXvrHXdVJ");
      const ethResponse =
        await client.BalanceService.getTokenBalancesForWalletAddress(
          "eth-sepolia",
          address
        );
      const polygonResponse =
        await client.BalanceService.getTokenBalancesForWalletAddress(
          "matic-mumbai",
          address
        );
      console.log(polygonResponse);
      if (this.eoaAddress === address) {
        this.eoaBalance = [
          {
            chain: "Ethereum Sepolia",
            data: ethResponse.data.items,
          },
          {
            chain: "Polygon Mumbai",
            data: polygonResponse.data.items,
          },
        ];
      } else if (this.avocadoAddress === address) {
        this.avocadoBalance = [
          {
            chain: "Ethereum Sepolia",
            data: ethResponse.data.items,
          },
          {
            chain: "Polygon Mumbai",
            data: polygonResponse.data.items,
          },
        ];
      }
    },
  },
};
</script>

<style>
body {
  background-color: rgb(66, 66, 66);
  color: aliceblue;
}
</style>
