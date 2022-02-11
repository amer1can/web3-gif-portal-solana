<template>
  <div class="main">
    <div class="header pt-4">GIF PORTAL</div>
    <p class="subheader mt-3">Collect your GIFs in metaverse</p>

    <div v-if="!walletAddress">
      <button class="wallet-btn mt-3" @click="connectToWallet">{{ walletStatus }}</button>
<!--      <img v-if="" class="d-block mx-auto mt-5" width="150" height="150" src="./assets/waiting.gif" alt="">-->
    </div>

    <div v-else class="portal-wrap">
      <div v-if="gifList === null" class="createAccBtn">
        <button class="btn btn-dark" @click="createGifAccount">Initialize GIF PORTAL account</button>
      </div>

      <div v-else class="gif-list">
        <form class="d-flex mb-5" @submit.prevent="sendGif">
          <input class="form-control" type="text" v-model="value" placeholder="Paste your GIF address here...">
          <button type="submit" class="btn btn-dark mx-3">Submit</button>
        </form>

        <div class="gif-wrapper">
          <div class="gif-card" v-for="(gif,idx) in gifList" :key="idx">
            <img :src="gif.gifLink" :title="gif.userAddress.toString()">
          </div>
        </div>
      </div>
    </div>


  </div>

  <div class="footer">
    <img src="https://gateway.pinata.cloud/ipfs/QmVYRRvrJRNNfhxnp12dW4zYMS5iXwavxCk8oRUPwhdDQk" alt="">
    <a href="https://twitter.com/amer1canWM" target="_blank" rel="noreferrer">Seeing on Twitter</a>
  </div>
</template>

<script>

/*https://giphy.com/ - more gifs
rust install: curl --proto '=https' --tlsv1.2 https://sh.rustup.rs -sSf | sh
solana install : sh -c "$(curl -sSfL https://release.solana.com/v1.9.5/install)"
      Please update your PATH environment variable to include the solana programs: copy and paste the recommended command below it to update PATH
      'solana-install update' may be used to easily update the Solana software to a newer version at any time.
next:
solana config set --url localhost
solana config get
solana-test-validator
if error: Unable to connect to validator: Client error: test-ledger/admin.rpc does not exist
try -> cd ~ to root -> cd home/admina

npm install -g mocha
install anchor: npm install --global yarn (if not install)
sudo apt-get update && sudo apt-get upgrade && sudo apt-get install -y pkg-config build-essential libudev-dev libssl-dev
cargo install --git https://github.com/project-serum/anchor anchor-cli --locked

or: sudo npm i -g @project-serum/anchor-cli @solana/web3.js
------------
anchor init proj --javascript
cd proj
solana-keygen new (solana address -> show address)
anchor test
\\wsl$\Ubuntu-20.04
*/

import  { Connection, PublicKey, clusterApiUrl } from '@solana/web3.js';
import { Program, Provider, web3 } from '@project-serum/anchor';
import idl from './idl.json';
import kp from './keypair.json';

const { SystemProgram } = web3; // SystemProgram is a reference to the Solana runtime!
const arr = Object.values(kp._keypair.secretKey);
const secret = new Uint8Array(arr);
const baseAccount = web3.Keypair.fromSecretKey(secret)

const programId = new PublicKey(idl.metadata.address); // Get our program's id from the IDL file.
const network = clusterApiUrl('devnet'); // Set our network to devnet.
const opts = { preflightCommitment: "processed" } // Controls how we want to acknowledge when a transaction is "done".



export default {
  name: 'App',
  data() {
    return {
      gifList: [],
      value: '',
      walletStatus: 'Connect to wallet',
      walletAddress: null,
    }
  },
  async mounted() {
    await this.checkIfWalletConnect()
    if(this.walletAddress) {
      console.log('Fetching GIF list...');
      await this.getGifList();
    }
  },
  computed: {},
  methods: {
    async checkIfWalletConnect() {
      try {
        const { solana } = window;
        if (solana) {
          if (solana.isPhantom) {
            console.log('Phantom wallet found!', solana)
            const response = await solana.connect({ onlyIfTrusted: true })
            console.log('Trusted Connection with Public Key:', response.publicKey.toString())
            this.walletAddress = response.publicKey.toString()
            this.walletStatus = 'Connected'
          }
        } else { alert('Solana object not found! Get a Phantom wallet!') }

      } catch (err) {
        console.log('CheckWalletConnection Error!', err)
      }
    },
    async connectToWallet() {
      const { solana } = window;
      if (solana) {
          const response = await solana.connect()
          console.log('Connected with Public Key:', response.publicKey.toString())
          this.walletAddress = response.publicKey.toString()
          this.walletStatus = 'Connected'
      }
    },
    getProvider() {
      const connection = new Connection(network, opts.preflightCommitment);
      const provider = new Provider(connection, window.solana, opts.preflightCommitment);
      return provider;
    },
    async createGifAccount() {
      try {
        const provider = this.getProvider();
        const program = new Program(idl, programId, provider);
        console.log('ping...');
        await program.rpc.initialize({
          accounts: {
            baseAccount: baseAccount.publicKey,
            user: provider.wallet.publicKey,
            systemProgram: SystemProgram.programId,
          },
          signers: [baseAccount]
        });
        console.log('Created a new BaseAccount with address:', baseAccount.publicKey.toString())
        await this.getGifList();
      } catch(err) {
        console.error('createGifAccount error', err)
      }
    },
    async sendGif() {
      if (this.value.length === 0) {
        console.log('No gif link given!')
        return;
      }
      console.log('Gif link:', this.value);
      try {
        const provider = this.getProvider();
        const program = new Program(idl, programId, provider);

        await program.rpc.addGif(this.value, {
          accounts: {
            baseAccount: baseAccount.publicKey,
            user: provider.wallet.publicKey
          },
        });
        console.log('GIF successfully sent to program', this.value)
        await this.getGifList()
        this.value = '';
      } catch(err) {
        console.log('sendGif error', err)
      }
    },
    async getGifList() {
      try {
        const provider = this.getProvider();
        const program = new Program(idl, programId, provider);
        const account = await program.account.baseAccount.fetch(baseAccount.publicKey);

        console.log('Got the account: ', account)
        this.gifList = account.gifList

      } catch(err) {
        console.error('getGifList error', err)
        this.gifList = null
      }
    }

  }

}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  background: black;
  display: flex;
  flex-direction: column;
  height: 100vh;
}
.main {
  flex-grow: 1;
}
.header {
  font-size: 50px;
  font-weight: bold;
  background: -webkit-linear-gradient(left, #6ff3da 30%, #135aee 60%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}
.subheader {
  font-size: 30px;
  color: white;
}
.wallet-btn {
  font-size: 22px;
  padding: 0.5em;
  border-radius: 15px;
  background: -webkit-linear-gradient(left, #6ff3da 30%, #135aee 60%);
  background-size: 200% 200%;
  animation: gradient-animation 4s ease infinite;
  align-self: center;
}
.portal-wrap {
  margin: 60px auto 0;
  max-width: 1200px;
}
.gif-wrapper {
  display: flex;
  flex-wrap: wrap;
}
.gif-card img {
  width: 150px;
  height: 150px;
  margin: 0px 10px;
  border-radius: 10px;
}

.footer {
  padding: 20px 0px;
}
.footer a {
  color: coral;
  font-size: 18px;
  font-weight: bold;
  text-decoration: none;
  transition: all 0.2s ease-in;
}
.footer a:hover {
  color: #ff4444;
  transition: all 0.2s ease-in;
}
.footer img {
  width: 30px;
}

/*------------------------------------------------------*/
/* TOAST */

@keyframes fadeout {
  from {
    top: 60px;
    opacity: 1;
  }
  to {
    top: 0px;
    opacity: 0;
  }
}


/* SHAKE */
.shake-slow {
  display: inherit;
  transform-origin: center center;
  animation-play-state: running
}
@keyframes shake-slow {
  2% {
    transform: translate(2px, 1px) rotate(-0.5deg); }
  4% {
    transform: translate(-5px, -9px) rotate(-2.5deg); }
  6% {
    transform: translate(5px, 3px) rotate(2.5deg); }
  8% {
    transform: translate(5px, 4px) rotate(1.5deg); }
  10% {
    transform: translate(-4px, -8px) rotate(-0.5deg); }
  12% {
    transform: translate(-2px, -9px) rotate(1.5deg); }
  14% {
    transform: translate(0px, 6px) rotate(3.5deg); }
  16% {
    transform: translate(8px, 2px) rotate(2.5deg); }
  18% {
    transform: translate(-4px, -1px) rotate(1.5deg); }
  20% {
    transform: translate(10px, 2px) rotate(-2.5deg); }
  22% {
    transform: translate(-2px, -4px) rotate(1.5deg); }
  24% {
    transform: translate(-8px, 5px) rotate(-0.5deg); }
  26% {
    transform: translate(-3px, -5px) rotate(3.5deg); }
  28% {
    transform: translate(6px, 2px) rotate(2.5deg); }
  30% {
    transform: translate(-5px, -2px) rotate(2.5deg); }
  32% {
    transform: translate(5px, -6px) rotate(-2.5deg); }
  34% {
    transform: translate(5px, -7px) rotate(-1.5deg); }
  36% {
    transform: translate(3px, -9px) rotate(0.5deg); }
  38% {
    transform: translate(6px, -2px) rotate(-0.5deg); }
  40% {
    transform: translate(-4px, -2px) rotate(0.5deg); }
  42% {
    transform: translate(7px, -8px) rotate(1.5deg); }
  44% {
    transform: translate(-4px, 10px) rotate(-1.5deg); }
  46% {
    transform: translate(-3px, 8px) rotate(-1.5deg); }
  48% {
    transform: translate(3px, 6px) rotate(-0.5deg); }
  50% {
    transform: translate(4px, -1px) rotate(-2.5deg); }
  52% {
    transform: translate(1px, 5px) rotate(-0.5deg); }
  54% {
    transform: translate(-3px, 7px) rotate(-2.5deg); }
  56% {
    transform: translate(3px, 3px) rotate(-2.5deg); }
  58% {
    transform: translate(-8px, -5px) rotate(-1.5deg); }
  60% {
    transform: translate(1px, -9px) rotate(3.5deg); }
  62% {
    transform: translate(0px, -3px) rotate(-0.5deg); }
  64% {
    transform: translate(4px, 2px) rotate(2.5deg); }
  66% {
    transform: translate(4px, 10px) rotate(1.5deg); }
  68% {
    transform: translate(1px, 2px) rotate(-2.5deg); }
  70% {
    transform: translate(-9px, 10px) rotate(0.5deg); }
  72% {
    transform: translate(-5px, -8px) rotate(0.5deg); }
  74% {
    transform: translate(-3px, -8px) rotate(3.5deg); }
  76% {
    transform: translate(-1px, 8px) rotate(3.5deg); }
  78% {
    transform: translate(-5px, -4px) rotate(0.5deg); }
  80% {
    transform: translate(-6px, 1px) rotate(1.5deg); }
  82% {
    transform: translate(-5px, -7px) rotate(1.5deg); }
  84% {
    transform: translate(1px, -6px) rotate(-0.5deg); }
  86% {
    transform: translate(-8px, -4px) rotate(3.5deg); }
  88% {
    transform: translate(-6px, 7px) rotate(-1.5deg); }
  90% {
    transform: translate(9px, 6px) rotate(-2.5deg); }
  92% {
    transform: translate(9px, -8px) rotate(3.5deg); }
  94% {
    transform: translate(3px, 3px) rotate(2.5deg); }
  96% {
    transform: translate(-6px, -7px) rotate(2.5deg); }
  98% {
    transform: translate(5px, -2px) rotate(-0.5deg); }
  0%, 100% {
    transform: translate(0, 0) rotate(0); } }
.shake-slow {
  animation-name: shake-slow;
  animation-duration: 5s;
  animation-timing-function: ease-in-out;
  animation-iteration-count: infinite;
}
/*------------------------------------------------------*/
.shake-little {
  display: inherit;
  transform-origin: center center;
  animation-play-state: running;
}

@keyframes shake-little {
  2% {
    transform: translate(0px, 1px) rotate(0.5deg); }
  4% {
    transform: translate(0px, 1px) rotate(0.5deg); }
  6% {
    transform: translate(1px, 1px) rotate(0.5deg); }
  8% {
    transform: translate(0px, 0px) rotate(0.5deg); }
  10% {
    transform: translate(1px, 0px) rotate(0.5deg); }
  12% {
    transform: translate(1px, 1px) rotate(0.5deg); }
  14% {
    transform: translate(0px, 0px) rotate(0.5deg); }
  16% {
    transform: translate(1px, 0px) rotate(0.5deg); }
  18% {
    transform: translate(0px, 1px) rotate(0.5deg); }
  20% {
    transform: translate(0px, 1px) rotate(0.5deg); }
  22% {
    transform: translate(1px, 1px) rotate(0.5deg); }
  24% {
    transform: translate(0px, 1px) rotate(0.5deg); }
  26% {
    transform: translate(0px, 0px) rotate(0.5deg); }
  28% {
    transform: translate(1px, 0px) rotate(0.5deg); }
  30% {
    transform: translate(1px, 0px) rotate(0.5deg); }
  32% {
    transform: translate(0px, 1px) rotate(0.5deg); }
  34% {
    transform: translate(0px, 1px) rotate(0.5deg); }
  36% {
    transform: translate(0px, 0px) rotate(0.5deg); }
  38% {
    transform: translate(1px, 1px) rotate(0.5deg); }
  40% {
    transform: translate(1px, 0px) rotate(0.5deg); }
  42% {
    transform: translate(0px, 1px) rotate(0.5deg); }
  44% {
    transform: translate(0px, 1px) rotate(0.5deg); }
  46% {
    transform: translate(0px, 0px) rotate(0.5deg); }
  48% {
    transform: translate(0px, 1px) rotate(0.5deg); }
  50% {
    transform: translate(0px, 1px) rotate(0.5deg); }
  52% {
    transform: translate(1px, 1px) rotate(0.5deg); }
  54% {
    transform: translate(0px, 0px) rotate(0.5deg); }
  56% {
    transform: translate(0px, 0px) rotate(0.5deg); }
  58% {
    transform: translate(0px, 0px) rotate(0.5deg); }
  60% {
    transform: translate(0px, 1px) rotate(0.5deg); }
  62% {
    transform: translate(0px, 0px) rotate(0.5deg); }
  64% {
    transform: translate(1px, 0px) rotate(0.5deg); }
  66% {
    transform: translate(0px, 1px) rotate(0.5deg); }
  68% {
    transform: translate(0px, 0px) rotate(0.5deg); }
  70% {
    transform: translate(1px, 1px) rotate(0.5deg); }
  72% {
    transform: translate(1px, 0px) rotate(0.5deg); }
  74% {
    transform: translate(0px, 1px) rotate(0.5deg); }
  76% {
    transform: translate(0px, 1px) rotate(0.5deg); }
  78% {
    transform: translate(1px, 0px) rotate(0.5deg); }
  80% {
    transform: translate(1px, 0px) rotate(0.5deg); }
  82% {
    transform: translate(0px, 0px) rotate(0.5deg); }
  84% {
    transform: translate(1px, 0px) rotate(0.5deg); }
  86% {
    transform: translate(0px, 0px) rotate(0.5deg); }
  88% {
    transform: translate(0px, 0px) rotate(0.5deg); }
  90% {
    transform: translate(1px, 1px) rotate(0.5deg); }
  92% {
    transform: translate(1px, 1px) rotate(0.5deg); }
  94% {
    transform: translate(1px, 0px) rotate(0.5deg); }
  96% {
    transform: translate(1px, 0px) rotate(0.5deg); }
  98% {
    transform: translate(0px, 0px) rotate(0.5deg); }
  0%, 100% {
    transform: translate(0, 0) rotate(0); } }

.shake-little {
  animation-name: shake-little;
  animation-duration: 100ms;
  animation-timing-function: ease-in-out;
  animation-iteration-count: infinite;
}

/* KeyFrames */
@-webkit-keyframes gradient-animation {
  0% {
    background-position: 0% 50%;
  }
  50% {
    background-position: 100% 50%;
  }
  100% {
    background-position: 0% 50%;
  }
}
@-moz-keyframes gradient-animation {
  0% {
    background-position: 0% 50%;
  }
  50% {
    background-position: 100% 50%;
  }
  100% {
    background-position: 0% 50%;
  }
}
@keyframes gradient-animation {
  0% {
    background-position: 0% 50%;
  }
  50% {
    background-position: 100% 50%;
  }
  100% {
    background-position: 0% 50%;
  }
}
</style>
