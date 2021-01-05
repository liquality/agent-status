<script>
import { onMount } from 'svelte'

import ALLOCATION from '../allocation.json'

const formatter = new Intl.NumberFormat()
const networks = ['', 'testnet']
const envs = ['', 'dev']

const currencyMap = {
  BTC: 1e8,
  ETH: 1e18,
  DAI: 1e18,
  USDC: 1e6,
  USDT: 1e6,
  WBTC: 1e8,
  UNI: 1e18,
  RBTC: 1e18
}
const currencyDecimalMapUI = {
  BTC: 1e6,
  ETH: 1e6,
  DAI: 1e2,
  USDC: 1e2,
  USDT: 1e2,
  WBTC: 1e6,
  UNI: 1e6,
  RBTC: 1e6
}
const results = {}
const totalUSD = {}
const USD = {
  BTC: 0,
  ETH: 0,
  DAI: 0,
  USDC: 0,
  USDT: 0,
  WBTC: 0,
  UNI: 0,
  RBTC: 0
}

const DROP = 0.2
const INCREASE = 0.2
const MINMAX = {}

Object.entries(ALLOCATION).forEach(([ asset, target ]) => {
  MINMAX[asset] = {
    min: target * (1 - DROP),
    target,
    max: target * (1 + INCREASE)
  }
})

const makeId = (network, env) => {
  if (!network) network = 'mainnet'
  if (!env) env = 'prod'

  return [network, env].join('-')
}

const makeUrl = (network, env) => {
  let arr = [network, env].filter(t => !!t).join('-')
  if (arr.length > 0) arr = '-' + arr
  return `https://liquality.io/swap${arr}/agent/api/swap/assetinfo`
}

const prettyCurrency = (code, value) => {
  const val = value / currencyMap[code]
  return Math.floor(val * currencyDecimalMapUI[code]) / currencyDecimalMapUI[code]
}

networks.forEach(network => {
  envs.forEach(env => {
    const id = makeId(network, env)
    totalUSD[id] = 0
    results[id] = []
  })
})

const random = (min, max) => Math.floor(Math.random() * (max - min + 1)) + min

async function run (network, env) {
  const response = await fetch(makeUrl(network, env))
  const assets = await response.json()
  const id = makeId(network, env)

  totalUSD[id] = 0
  results[id] = assets.map(({ code, actualBalance }) => {
    actualBalance = prettyCurrency(code, actualBalance)
    const usd = network === 'testnet' ? 0 : Math.floor(USD[code] * actualBalance * 100) / 100

    totalUSD[id]+= usd

    return {
      code,
      actualBalance,
      usd
    }
  })

  results[id] = results[id].map(asset => {
    asset.share = Math.floor((asset.usd / totalUSD[id]) * 100) || 0
    const mm = MINMAX[asset.code]

    if (asset.share >= mm.min && asset.share <= mm.max) {
      asset.level = 'success'
    } else {
      asset.level = 'danger'
    }

    asset.diff = Math.floor(asset.usd - ((mm.target / 100) * totalUSD[id]))
    asset.diff2 = prettyCurrency(asset.code, (asset.diff / USD[asset.code]) * currencyMap[asset.code])

    if (asset.diff > 0) {
      asset.level2 = 'success'
    } else {
      asset.level2 = 'danger'
    }

    asset.diff = `${asset.diff}`.replace('-', '')
    asset.diff2 = `${asset.diff2}`.replace('-', '')

    return asset
  })

  setTimeout(run, random(30000, 60000), network, env)
}

async function updateUSD () {
  const response = await fetch('https://api.coingecko.com/api/v3/simple/price?ids=ethereum,bitcoin,dai,usd-coin,tether,wrapped-bitcoin,uniswap,rootstock&vs_currencies=usd')
  const assets = await response.json()

  USD.BTC = assets.bitcoin.usd
  USD.ETH = assets.ethereum.usd
  USD.DAI = assets.dai.usd
  USD.USDC = assets['usd-coin'].usd
  USD.USDT = assets.tether.usd
  USD.WBTC = assets['wrapped-bitcoin'].usd
  USD.UNI = assets.uniswap.usd
  USD.RBTC = assets.rootstock.usd

  setTimeout(updateUSD, random(15000, 60000))
}

async function start () {
  await updateUSD()

  networks.forEach(network => {
    envs.forEach(env => run(network, env))
  })
}

onMount(start)
</script>

<main>
	<div class="container">
  <ul class="list-inline">
    <li class="list-inline-item"><span class="small text-muted">TARGET</span></li>
  {#each Object.entries(MINMAX) as [code, mm]}
    <li class="list-inline-item">{code} &mdash; {mm.target}%</li>
  {/each}
  </ul>
  {#each Object.entries(results) as [id, assets]}
    <h1 class="h4">
      <span>{id}</span>
      <span class="small font-weight-light ml-3">{formatter.format(Math.floor(totalUSD[id] * 100) / 100)} USD</span>
    </h1>
		<div class="row mb-4">
    {#each assets as asset}
  		<div class="col-lg-3 col-md-3 col-12 mb-4">
  			<div class="card">
  			  <div class="card-body">
  			    <h5 class="card-title">
  						<span>{asset.code}</span>
              <small class="badge badge-{asset.level2}">
                {asset.diff2} &mdash; {formatter.format(asset.diff)} USD
              </small>
  					</h5>
  			    <h6 class="h2 font-weight-light" title="{formatter.format(asset.actualBalance)} {asset.code}">{formatter.format(asset.actualBalance)}</h6>
            <p class="card-text">
              <span class="small text-muted">~${formatter.format(asset.usd)} USD</span>
              <small class="badge badge-{asset.level} font-weight-light">{asset.share}%</small>
            </p>
  			  </div>
  			</div>
  		</div>
    {/each}
		</div>
  {/each}
	</div>
</main>

<style>
main {
	height: 100%;
	display: flex;
	justify-content: center;
  align-items: center;
}

.card-title {
	display: flex;
	justify-content: space-between;
	align-items: center;
}

h1.h4 {
  display: flex;
	align-items: center;
}

h6.h2 {
  word-break: break-all;
  text-overflow: ellipsis;
  white-space: nowrap;
  overflow: hidden;
}

.card-title small {
	font-size: 0.85rem;
	font-weight: 400;
}

.list-inline-item + .list-inline-item {
  margin-left: 1rem;
}

.card-text {
  display: flex;
  justify-content: space-between;
	align-items: center;
}
</style>
