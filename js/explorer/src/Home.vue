<template>
<div>
    <b-navbar toggleable="md" type="dark" variant="info">
    <b-container>
        <b-navbar-brand>Matter Network</b-navbar-brand>
        <b-navbar-toggle target="nav-collapse"></b-navbar-toggle>
        <b-collapse id="nav-collapse" is-nav>
        <b-navbar-nav>
            <b-nav-item href="/client/" target="_blanc">MatterMask</b-nav-item>
            <b-nav-item v-bind:href="`${etherscan}/address/0x${store.config.CONTRACT_ADDR}`" target="_blanc">
                Contract <span style="font-size: 0.9em"><i class="fas fa-external-link-alt"></i></span>
            </b-nav-item>
        </b-navbar-nav>
        <b-navbar-nav class="ml-auto">
            <b-nav-item-dropdown :text="store.network" class="capitalize" right>
                <b-dropdown-item href="https://mainnet.matter-labs.io" target="blanc">Mainnet</b-dropdown-item>
                <b-dropdown-item href="https://rinkeby.matter-labs.io" target="blanc">Rinkeby</b-dropdown-item>
            </b-nav-item-dropdown>
        </b-navbar-nav>
        </b-collapse>
    </b-container>
    </b-navbar>
    <br>
    <b-container>
        <b-card bg-variant="light" >
            <h4>Matter Testnet Block Explorer</h4> 
            <b-form @submit.stop.prevent="search">
            <b-input-group>
                <b-form-input v-model="query" placeholder="block number, tx hash or state root hash"></b-form-input>
                <b-input-group-append>
                <b-button @click="search" variant="info" :disabled="searching">
                    <b-spinner v-if="searching" small></b-spinner>
                    <span>Search</span>
                </b-button>
                </b-input-group-append>
                <b-form-invalid-feedback v-if="notFound" :state="false">
                    Nothing found for query '{{query}}'.
                </b-form-invalid-feedback>
            </b-input-group>
            </b-form>
        </b-card>
        <br>
        <b-card>
        <div class="row" style="color: grey">
            <div class="col-sm text-center">
            <i class="far fa-square"></i> <b>Blocks committed</b><br><span class="num">{{lastCommitted}}</span>
            </div>
            <div class="col-sm text-center">
            <i class="far fa-check-square"></i> <b>Blocks verified</b><br><span class="num">{{lastVerified}}</span>
            </div>
            <div class="col-sm text-center">
            <i class="fas fa-list"></i> <b>Total transactions</b><br><span class="num">{{totalTransactions}}</span>
            </div>
            <div class="col-sm text-center">
            <i class="fas fa-archive"></i> <b>Tx per block</b><br><span class="num">{{txPerBlock}}</span>
            </div>
        </div>
        </b-card>
        <br>

        <b-pagination v-if="ready" v-model="currentPage" :per-page="perPage" :total-rows="rows" @change="onPageChanged"></b-pagination>
        <b-table responsive id="table" hover outlined :items="items" @row-clicked="onRowClicked" :busy="loading" class="clickable"></b-table>
        <b-pagination v-if="ready" v-model="currentPage" :per-page="perPage" :total-rows="rows" @change="onPageChanged"></b-pagination>

    </b-container>
</div>
</template>

<style>

.capitalize:first-letter {
    text-transform: capitalize;
}

.table-container {
  position: relative;
}

.overlay {
  position: absolute;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
}

.clickable tr {
    cursor: pointer;
}

.num {
    font-size: 3em;
}

@media (max-width: 720px) {
.hide-sm {
    display: none
}
}

@media (max-width: 992px) {
.hide-lg {
    display: none
}
}

h1, h2, h3, h4 {
    font-weight: bold;
}

body {
    font-size: 0.9rem;
}

.btn {
    font-size: 0.8rem;
}

</style>

<script>

import store from './store'
import client from './client'

export default {
    name: 'home',
    created() {
        this.update()
    },
    timers: {
        ticker: { time: 1000, autostart: true, repeat: true }
    },
    methods: {
        ticker() {
            this.update(true)
        },
        async search() {
            if (this.query) {
                this.searching = true
                this.notFound = false
                let block = await client.searchBlock(this.query)
                this.searching = false
                if (block && block.block_number) {
                    this.$router.push('/blocks/' + block.block_number)
                } else {
                    this.notFound = true
                    await new Promise(resolve => setTimeout(resolve, 3600))
                    this.notFound = false
                }
            }
        },
        onRowClicked(item) {
            this.$router.push('/blocks/' + item.block_number)
        },
        async onPageChanged(page) {
            this.$router.push(`${this.$route.path}?page=${page}`)
            //this.updateBlocks()
        },
        async update(silent) {
            if (!silent) {
                this.loading = true
            }
            const status = await client.status()
            let newBlocks = false
            if (status) {
                newBlocks = this.lastCommitted !== status.last_committed || this.lastVerified !== status.last_verified
                this.lastCommitted = status.last_committed
                this.lastVerified = status.last_verified
                this.totalTransactions = status.total_transactions
            }
            if (newBlocks) {
                this.updateBlocks()
            } else {
                this.loading = false
            }
        },
        async updateBlocks() {
            let max = this.lastCommitted - (client.PAGE_SIZE * (this.currentPage-1))
            if (max < 0) return

            let blocks = await client.loadBlocks(max)
            if (blocks) {
                this.blocks = blocks.map( b => ({
                    block_number:   b.block_number,
                    status:         b.verified_at ? 'Verified' : 'Committed',
                    new_state_root: b.new_state_root.slice(0, 16) + '...' + b.new_state_root.slice(50, 66),
                    committed_at:   b.committed_at,
                    verified_at:    b.verified_at,
                }))
                this.currentPage = this.page
                this.ready = true
            }
            this.loading = false
        },
    },
    watch: {
        '$route' (to, from) {
            this.currentPage = this.page
            this.updateBlocks()
        },
    },
    computed: {
        page() {
            return this.$route.query.page || 1
        },
        items() {
            return this.blocks
        },
        perPage() {
            return client.PAGE_SIZE
        },
        rows() {
            return this.lastCommitted || 9999
        },
    },
    data() {
        return {
            lastCommitted:      0,
            lastVerified:       0,
            totalTransactions:  0,
            currentPage:        this.$route.query.page || 1,
            
            txPerBlock:         client.TX_PER_BLOCK(),
            blocks:             [],
            ready:              false,

            query:              '',
            loading:            true,
            searching:          false,
            notFound:           false,

            breadcrumbs: [
                {
                    text: 'Blocks',
                    active: true
                },
            ],
        }
    },
}
</script>
