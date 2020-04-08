<script>
import { onMount } from 'svelte'

const formatter = new Intl.NumberFormat()
const networks = ['', 'testnet']
const envs = ['', 'dev']

const currencyMap = {
  BTC: 1e8,
  ETH: 1e18,
  DAI: 1e18,
  USDC: 1e6
}
const currencyDecimalMap = {
  BTC: 1e6,
  ETH: 1e6,
  DAI: 1e2,
  USDC: 1e2
}
const results = {}
const USD = {
  BTC: 0,
  ETH: 0,
  DAI: 0,
  USDC: 0
}

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
  return Math.floor(val * currencyDecimalMap[code]) / currencyDecimalMap[code]
}

networks.forEach(network => {
  envs.forEach(env => {
    results[makeId(network, env)] = []
  })
})

const random = (min, max) => Math.floor(Math.random() * (max - min + 1)) + min

async function updateUSD () {
  const response = await fetch('https://api.coingecko.com/api/v3/simple/price?ids=ethereum,bitcoin,dai,usd-coin&vs_currencies=usd')
  const assets = await response.json()

  USD.BTC = assets.bitcoin.usd
  USD.ETH = assets.ethereum.usd
  USD.DAI = assets.dai.usd
  USD.USDC = assets['usd-coin'].usd

  setTimeout(run, random(30000, 60000))
}

async function run (network, env) {
	const response = await fetch(makeUrl(network, env))
  const assets = await response.json()

  results[makeId(network, env)] = assets.map(({ code, actualBalance }) => {
    actualBalance = prettyCurrency(code, actualBalance)

    return {
      code,
      actualBalance
    }
  })

  setTimeout(run, random(15000, 60000), network, env)
}

function start () {
  updateUSD()

  networks.forEach(network => {
    envs.forEach(env => run(network, env))
  })
}

onMount(start)
</script>

<main>
	<div class="container">
  {#each Object.entries(results) as [id, assets]}
    <h1 class="h4">{id}</h1>
		<div class="row mb-4">
    {#each assets as asset}
  		<div class="col-lg-3 col-md-3 col-12 mb-4">
  			<div class="card">
  			  <div class="card-body">
  			    <h5 class="card-title">
  						<span>{asset.code}</span>
  					</h5>
  			    <h6 class="h2 font-weight-light" title="{formatter.format(asset.actualBalance)}">{formatter.format(asset.actualBalance)}</h6>
            <p class="card-text small text-muted">~${formatter.format(Math.floor(USD[asset.code] * asset.actualBalance * 100) / 100)} USD</p>
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
</style>
