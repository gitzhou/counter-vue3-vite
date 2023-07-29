<script setup lang="ts">

import { Counter } from './contracts/counter';
import { Scrypt, ScryptProvider, SensiletSigner, ContractCalledEvent } from 'scrypt-ts';
import { ref, onMounted } from 'vue'
import type { Ref } from 'vue'

const contractId = {
  txId: 'fdc5218259b0d8b2127873537339d58b7c8c1e19f0859b6bc3d3549d48d64a3c',
  outputIndex: 0,
}

const instance: Ref<Counter | null> = ref(null)
const counterText: Ref<string> = ref('null')
const signerRef = ref(new SensiletSigner(new ScryptProvider()))

async function updateInstance(newInstance: Counter | null = null) {
  if (!newInstance) {
    try {
      newInstance = await Scrypt.contractApi.getLatestInstance(Counter, contractId)
    } catch (e: any) {
      console.log(`Fetch instance error: ${e}`)
    }
  }
  instance.value = newInstance
  counterText.value = newInstance!.count.toString()
}

onMounted(async () => {
  updateInstance()
  Scrypt.contractApi.subscribe({
    clazz: Counter,
    id: contractId,
  }, (event: ContractCalledEvent<Counter>) => {
    const txId = event.tx.id
    console.log(`Counter increment: ${txId}`)
    updateInstance(event.nexts[0])
  })
})

const increment = async () => {
  const signer = signerRef.value as SensiletSigner
  if (instance.value && signer) {
    const { isAuthenticated, error } = await signer.requestAuth()
    if (!isAuthenticated) {
      throw new Error(error)
    }

    const currentInstance = instance.value
    await currentInstance.connect(signer)

    const nextInstance = currentInstance.next()
    nextInstance.count++

    currentInstance.methods.increment({
      next: {
        instance: nextInstance,
        balance: currentInstance.balance
      }
    }).catch(e => {
      console.log(`Counter call error: ${e}`)
      updateInstance()
    })
  }
}

</script>

<template>
  <header>
    <img alt="Vue logo" class="logo" src="./assets/logo.svg" width="125" height="125" />
  </header>

  <main>
    <h1>Counter now is {{ counterText }}</h1>
    <button @click="increment">
      Increment
    </button>
  </main>
</template>

<style scoped>
header {
  line-height: 1.5;
}

.logo {
  display: block;
  margin: 0 auto 2rem;
}

@media (min-width: 1024px) {
  header {
    display: flex;
    place-items: center;
    padding-right: calc(var(--section-gap) / 2);
  }

  .logo {
    margin: 0 2rem 0 0;
  }

  header .wrapper {
    display: flex;
    place-items: flex-start;
    flex-wrap: wrap;
  }
}
</style>
