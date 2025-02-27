<template lang="html">
	<div>
		<div class="page-header page-bar">
			<h1>{{ $t('title') }}</h1>
			<div class="tabs">
				<router-link v-if="env.BANK" to="/bank">
					<div class="tab action" icon="account_balance" link="/bank">
						<v-icon>mdi-bank</v-icon>
						<span>{{ $t('main.bank') }}</span>
					</div>
				</router-link>
				<div class="tab action active" image="icon/market.png" link="/market">
					<img src="/image/icon/black/market.png">
					<span>{{ $t('main.market') }}</span>
				</div>
			</div>
		</div>
		<div class="container last">
			<div v-show="!LeekWars.mobile || !LeekWars.splitBack" class="column8">
				<panel :title="$t('fights')" icon="mdi-sword-cross">
					<loader v-if="!fight_packs.length" slot="content" />
					<div v-else slot="content" class="items fights">
						<router-link v-for="pack in fight_packs" :key="pack.id" v-ripple :to="'/market/' + pack.name" :farmer-count="0" :leek-count="0" class="item fight-pack" @click="selectItem(pack)">
							<img :src="'/image/market/fight_pack_' + pack.fights + '.png'">
							<div>{{ pack.title }}</div>
						</router-link>
					</div>
				</panel>
				<panel :title="$t('weapons') + ' [' + weapons.length + ']'" icon="mdi-pistol">
					<loader v-if="!weapons.length" slot="content" />
					<div v-else slot="content" class="items weapons">
						<router-link v-for="weapon in weapons" :key="weapon.id" v-ripple :to="'/market/' + weapon.name.replace('weapon_', '')" :farmer-count="items[weapon.id].farmer_count" :leek-count="items[weapon.id].leek_count" class="item weapon" :class="{toohigh: weapon.level > max_level}">
							<img :src="'/image/' + weapon.name.replace('_', '/') + '.png'">
						</router-link>
					</div>
				</panel>
				<panel :title="$t('chips') + ' [' + chips.length + ']'" icon="mdi-chip">
					<div slot="actions" class="button flat" @click="updateChipMode">
						<v-icon v-if="chipMode === 'type'">mdi-sort-descending</v-icon>
						<v-icon v-else>mdi-view-grid</v-icon>
					</div>
					<loader v-if="!chips.length" slot="content" />
					<div v-else-if="chipMode === 'level'" slot="content" class="items chips">
						<router-link v-for="chip in chips" :key="chip.id" v-ripple :to="'/market/' + chip.name" :farmer-count="items[chip.id].farmer_count" :leek-count="items[chip.id].leek_count" class="item chip" :class="{toohigh: chip.level > max_level}">
							<img :src="'/image/chip/' + chip.name + '.png'">
						</router-link>
					</div>
					<div v-else slot="content">
						<div v-for="type in EffectTypeMarket" v-if="!isNaN(type)" :key="type">
							<h4 :class="{first: type === EffectTypeMarket.ATTACK}">{{ $t('effect.effect_type_' + type) }}</h4>
							<div class="items chips">
								<router-link v-for="chip in chipsByType[type]" :key="chip.id" v-ripple :to="'/market/' + chip.name" :farmer-count="items[chip.id].farmer_count" :leek-count="items[chip.id].leek_count" class="item chip" :class="{toohigh: chip.level > max_level}">
									<img :src="'/image/chip/' + chip.name + '.png'">
								</router-link>
							</div>
						</div>
					</div>
				</panel>
				<panel :title="$t('potions') + ' [' + potions.length + ']'" icon="mdi-bottle-tonic-plus-outline">
					<loader v-if="!potions.length" slot="content" />
					<div v-else slot="content" class="items potions">
						<router-link v-for="potion in potions" :key="potion.id" v-ripple :to="'/market/' + potion.name" :farmer-count="items[potion.id].farmer_count" :leek-count="items[potion.id].leek_count" class="item potion" :class="{toohigh: potion.level > max_level}">
							<img :src="'/image/potion/' + potion.name + '.png'">
						</router-link>
					</div>
				</panel>
				<panel :title="$t('hats') + ' [' + hats.length + ']'" icon="mdi-hat-fedora">
					<loader v-if="!hats.length" slot="content" />
					<div v-else slot="content" class="items hats">
						<router-link v-for="hat in hats" :key="hat.id" v-ripple :to="'/market/' + hat.name" :farmer-count="items[hat.id].farmer_count" :leek-count="items[hat.id].leek_count" class="item hat" :class="{toohigh: hat.level > max_level}">
							<img :src="'/image/hat/' + hat.name + '.png'">
						</router-link>
					</div>
				</panel>
				<panel :title="$t('pomps') + ' [' + pomps.length + ']'" icon="mdi-auto-fix">
					<loader v-if="!pomps.length" slot="content" />
					<div v-else slot="content" class="items pomps">
						<router-link v-for="pomp in pomps" :key="pomp.id" :to="'/market/' + pomp.name" :farmer-count="items[pomp.id].farmer_count" :leek-count="items[pomp.id].leek_count" class="item pomp" :class="{toohigh: pomp.level > max_level}">
							<img :src="'/image/pomp/' + pomp.name + '.png'">
						</router-link>
					</div>
				</panel>
			</div>
			<div v-show="!LeekWars.mobile || LeekWars.splitBack" class="column4">
				<panel :title="$t('characteristic.characteristics')" class="preview-panel" icon="mdi-information-outline">
					<loader v-if="!selectedItem" slot="content" />
					<div v-else slot="content" class="preview center">
						<weapon-preview v-if="selectedItem.type == ItemType.WEAPON" :weapon="LeekWars.weapons[LeekWars.items[selectedItem.id].params]" />
						<chip-preview v-else-if="selectedItem.type == ItemType.CHIP" :chip="LeekWars.chips[selectedItem.id]" />
						<potion-preview v-else-if="selectedItem.type == ItemType.POTION && LeekWars.potions[selectedItem.id]" :potion="LeekWars.potions[selectedItem.id]" />
						<fight-pack-preview v-else-if="selectedItem.type == ItemType.FIGHT_PACK" :pack="selectedItem" />
						<hat-preview v-else-if="selectedItem.type === ItemType.HAT && LeekWars.hats[selectedItem.id]" :hat="LeekWars.hats[selectedItem.id]" />
						<pomp-preview v-else-if="selectedItem.type == ItemType.POMP" :pomp="LeekWars.pomps[selectedItem.id]" />

						<router-link v-if="selectedItem.trophy" :to="'/trophy/' + selectedItem.trophy.name" class="trophy">
							<img :src="'/image/trophy/' + selectedItem.trophy.name + '.svg'">
							<i18n path="unlocked_with">
								<b slot="trophy">{{ $t('trophy.' + selectedItem.trophy.name) }}</b>
							</i18n>
						</router-link>

						<div class="buy-buttons">
							<div v-if="!selectedItem.buyable && !selectedItem.buyable_crystals" class="already-have">
								{{ $t('cannot_buy') }}
							</div>
							<div v-if="selectedItem.buyable">
								<h4 class="buy-label">{{ $t('buy') }}</h4>
								<v-btn :disabled="($store.state.farmer && $store.state.farmer.habs < selectedItem.price) || (selectedItem.singleton && (selectedItem.farmer_count > 0 || selectedItem.leek_count > 0))" class="buy-button" @click="openBuyHabs">{{ selectedItem.price | number }}<img src="/image/hab.png"></v-btn>
							</div>
							<div v-if="env.BANK && selectedItem.buyable_crystals">
								<h4 class="buy-label">{{ $t('buy') }}</h4>
								<v-btn :disabled="($store.state.farmer && $store.state.farmer.crystals < selectedItem.crystals) || (selectedItem.singleton && (selectedItem.farmer_count > 0 || selectedItem.leek_count > 0))" class="buy-crystals-button" @click="openBuyCrystals">{{ selectedItem.crystals | number }}<img src="/image/crystal.png"></v-btn>
							</div>
							<div v-if="selectedItem.singleton && (selectedItem.farmer_count > 0 || selectedItem.leek_count > 0)" class="already-have">
								{{ $t('already_have') }}
							</div>
							<template v-if="selectedItem.sellable && selectedItem.farmer_count > 0">
								<div class="sell">
									<h4 class="buy-label">{{ $t('resell') }}</h4>&nbsp;
									<v-btn class="sell-button" @click="sellDialog = true">{{ selectedItem.sell_price | number }} <img src="/image/hab.png"></v-btn>
								</div>
							</template>
						</div>
						<div v-if="selectedItem.leek_objs && selectedItem.leek_objs.length">
							<div><b>{{ $t('equipped_on') }}</b></div>
							<div class="leeks">
								<router-link v-for="leek in selectedItem.leek_objs" :key="leek.id" :to="'/leek/' + leek.id">
									<rich-tooltip-leek :id="leek.id" v-slot="{ on }">
										<div class="leek" v-on="on">{{ leek.name }}</div>
									</rich-tooltip-leek>
								</router-link>
							</div>
						</div>
					</div>
				</panel>

				<div v-if="!LeekWars.mobile" class="menu">
					<div v-ripple class="item" @click="scroll(0)">
						<v-icon>mdi-sword-cross</v-icon> {{ $t('fights') }}
					</div>
					<div v-ripple class="item" @click="scroll(1)">
						<v-icon>mdi-pistol</v-icon> {{ $t('weapons') }}
					</div>
					<div v-ripple class="item" @click="scroll(2)">
						<v-icon>mdi-chip</v-icon> {{ $t('chips') }}
					</div>
					<div v-ripple class="item" @click="scroll(3)">
						<v-icon>mdi-bottle-tonic-plus-outline</v-icon> {{ $t('potions') }}
					</div>
					<div v-ripple class="item" @click="scroll(4)">
						<v-icon>mdi-hat-fedora</v-icon> {{ $t('hats') }}
					</div>
					<div v-ripple class="item" @click="scroll(5)">
						<v-icon>mdi-auto-fix</v-icon> {{ $t('pomps') }}
					</div>
				</div>
			</div>
		</div>


		<div class="page-footer page-bar">
			<div class="tabs">
				<router-link to="/help/items" class="tab">
					<v-icon>mdi-chart-timeline-variant</v-icon>
					<span class="report-button">Progression des items</span>
				</router-link>
			</div>
		</div>

		<popup v-model="buyDialog" :width="600">
			<v-icon slot="icon">mdi-cash-multiple</v-icon>
			<span slot="title">{{ $t('confirm_purchase') }}</span>
			<div v-if="selectedItem && $store.state.farmer">
				<i18n tag="div" path="are_you_sure_you_want_to_buy">
					<b slot="item">{{ translateName(selectedItem) }}</b>
				</i18n>
				<br>
				<b>{{ $t('price') }}</b> : {{ selectedItem.price | number }} <span class="hab"></span>
				<br>
				<b>{{ $t('habs_before_purchase') }}</b> : {{ $store.state.farmer.habs | number }} <span class="hab"></span>
				<br>
				<b>{{ $t('habs_after_purchase') }}</b> : {{ $store.state.farmer.habs - selectedItem.price | number }} <span class="hab"></span>
			</div>
			<div slot="actions">
				<div v-ripple @click="buyDialog = false">{{ $t('cancel') }}</div>
				<div v-ripple class="buy green" @click="buy('habs')">{{ $t('buy') }}</div>
			</div>
		</popup>

		<popup v-model="buyCrystalsDialog" :width="600">
			<v-icon slot="icon">mdi-cash-multiple</v-icon>
			<span slot="title">{{ $t('confirm_purchase') }}</span>
			<div v-if="selectedItem && $store.state.farmer">
				<i18n tag="div" path="are_you_sure_you_want_to_buy">
					<b slot="item">{{ translateName(selectedItem) }}</b>
				</i18n>
				<br>
				<b>{{ $t('price') }}</b> : {{ selectedItem.crystals }} <span class="crystal"></span>
				<br>
				<b>{{ $t('crystals_before_purchase') }}</b> : {{ $store.state.farmer.crystals | number }} <span class="crystal"></span>
				<br>
				<b>{{ $t('crystals_after_purchase') }}</b> : {{ $store.state.farmer.crystals - selectedItem.crystals | number }} <span class="crystal"></span>
			</div>
			<div slot="actions">
				<div v-ripple @click="buyCrystalsDialog = false">{{ $t('cancel') }}</div>
				<div v-ripple class="buy green" @click="buy('crystals')">{{ $t('buy') }}</div>
			</div>
		</popup>

		<popup v-model="sellDialog" :width="600">
			<v-icon slot="icon">mdi-cash-multiple</v-icon>
			<span slot="title">{{ $t('confirm_sell') }}</span>
			<div v-if="selectedItem && $store.state.farmer">
				<i18n tag="div" path="are_you_sure_you_want_to_sell">
					<b slot="item">{{ translateName(selectedItem) }}</b>
				</i18n>
				<br>
				<b>{{ $t('price') }}</b> : {{ selectedItem.price | number }} <span class="hab"></span>
				<br>
				<b>{{ $t('habs_before_sell') }}</b> : {{ $store.state.farmer.habs | number }} <span class="hab"></span>
				<br>
				<b>{{ $t('habs_after_sell') }}</b> : {{ $store.state.farmer.habs + selectedItem.sell_price | number }} <span class="hab"></span>
			</div>
			<div slot="actions">
				<div v-ripple @click="sellDialog = false">{{ $t('cancel') }}</div>
				<div v-ripple class="sell red" @click="sell">{{ $t('sell') }}</div>
			</div>
		</popup>

		<popup v-model="unseenItemDialog" :width="400" :full="true">
			<v-icon slot="icon">mdi-new-box</v-icon>
			<span slot="title">{{ $t('new_item_unlocked') }}</span>
			<div v-if="unseenItem" class="unseen-dialog">
				<weapon-preview v-if="unseenItem.type == ItemType.WEAPON" :weapon="LeekWars.weapons[LeekWars.items[unseenItem.id].params]" />
				<chip-preview v-else-if="unseenItem.type == ItemType.CHIP" :chip="LeekWars.chips[unseenItem.id]" />
				<potion-preview v-else-if="unseenItem.type == ItemType.POTION" :potion="LeekWars.potions[unseenItem.id]" />
				<hat-preview v-else-if="unseenItem.type == ItemType.HAT" :hat="LeekWars.hats[unseenItem.id]" />
				<pomp-preview v-else-if="unseenItem.type == ItemType.POMP" :hat="LeekWars.pomps[unseenItem.id]" />

				<div v-if="unseenItem.trophy" class="card trophy">
					<img :src="'/image/trophy/' + unseenItem.trophy.name + '.svg'">
					<i18n path="unlocked_with">
						<b slot="trophy">{{ $t('trophy.' + unseenItem.trophy.name) }}</b>
					</i18n>
				</div>
			</div>
		</popup>
	</div>
</template>

<script lang="ts">
	import { ChipTemplate } from '@/model/chip'
	import { EffectTypeMarket } from '@/model/effect'
	import { Farmer } from '@/model/farmer'
	import { HatTemplate } from '@/model/hat'
	import { mixins } from '@/model/i18n'
	import { ItemTemplate, ItemType } from '@/model/item'
	import { LeekWars } from '@/model/leekwars'
	import { PompTemplate } from '@/model/pomp'
	import { PotionTemplate } from '@/model/potion'
	import { store } from '@/model/store'
	import { WeaponTemplate } from '@/model/weapon'
	import { Component, Vue, Watch } from 'vue-property-decorator'
	import ChipPreview from './chip-preview.vue'
	import FightPackPreview from './fight-pack-preview.vue'
	import HatPreview from './hat-preview.vue'
	import PompPreview from './pomp-preview.vue'
	import PotionPreview from './potion-preview.vue'
	import WeaponPreview from './weapon-preview.vue'

	@Component({
		name: 'market', i18n: {}, mixins,
		components: {
			'weapon-preview': WeaponPreview,
			'chip-preview': ChipPreview,
			'potion-preview': PotionPreview,
			'fight-pack-preview': FightPackPreview,
			'hat-preview': HatPreview,
			'pomp-preview': PompPreview
		}
	})
	export default class Market extends Vue {
		selectedItem: ItemTemplate | null = null
		items: {[key: string]: ItemTemplate} = {}
		weapons: ItemTemplate[] = []
		chips: ChipTemplate[] = []
		chipsByType: ChipTemplate[][] = []
		potions: PotionTemplate[] = []
		hats: HatTemplate[] = []
		items_by_name: {[key: string]: ItemTemplate} = {}
		fight_packs: any[] = []
		ItemType = ItemType
		buyDialog: boolean = false
		buyCrystalsDialog: boolean = false
		sellDialog: boolean = false
		chipMode: string = localStorage.getItem('market/sort_mode') === 'type' ? 'type' : 'level'
		EffectTypeMarket = EffectTypeMarket
		actions: any
		unseen_items: ItemTemplate[] = []
		unseenItem: ItemTemplate | null = null
		unseenItemDialog: boolean = false
		pomps: PompTemplate[] = []

		get max_level() {
			if (store.state.farmer) {
				return LeekWars.first(store.state.farmer.leeks)!.level
			}
			return 0
		}

		created() {
			this.actions = [{icon: 'mdi-bank', click: () => this.$router.push('/bank')}]
			LeekWars.get('market/get-item-templates').then(res => {
				const items = res.items as ItemTemplate[]
				for (const i in items) {
					const item = items[i]
					this.items[item.id] = item
					if (item.type === ItemType.WEAPON) {
						const w = LeekWars.items[item.id]
						this.weapons.push(w)
						this.items_by_name[w.name.replace('weapon_', '')] = item
					} else if (item.type === ItemType.CHIP) {
						const chip = LeekWars.chips[item.id]
						this.chips.push(chip)
						this.items_by_name[LeekWars.chips[item.id].name] = item
						// Place the chip in the categories which correspond to its effects
						for (const effect of chip.effects) {
							if (this.chipsByType[effect.type] === undefined) {
								this.chipsByType[effect.type] = []
							}
							this.chipsByType[effect.type].push(chip)
							break
						}
					} else if (item.type === ItemType.POTION) {
						const potion = LeekWars.potions[item.id]
						if (potion) {
							this.potions.push(potion)
							this.items_by_name[LeekWars.potions[item.id].name] = item
						} else {
							const fakePotion = {...item, name: item.name.replace(/^potion_/, ''), level: 1, consumable: 0, effects: [], template: item.id, duration: 0}
							this.potions.push(fakePotion)
							this.items_by_name[fakePotion.name] = fakePotion
						}
					} else if (item.type === ItemType.HAT) {
						const hat = LeekWars.hats[item.id]
						if (hat) {
							this.hats.push(hat)
							this.items_by_name[hat.name] = item
						} else {
							const fakeHat = {...item, name: item.name.replace(/^hat_/, ''), level: 1, width: 0, height: 0, template: item.id}
							this.hats.push(fakeHat)
							this.items_by_name[fakeHat.name] = fakeHat
						}
					} else if (item.type === ItemType.POMP) {
						this.pomps.push(LeekWars.pomps[item.id])
						this.items_by_name[LeekWars.pomps[item.id].name] = item
					}
					item.leek_objs = []
					if (this.$store.state.farmer) {
						for (const leek of item.leeks!) {
							if (leek in this.$store.state.farmer.leeks) {
								item.leek_objs.push(this.$store.state.farmer.leeks[leek])
							}
						}
					}
					if (!item.seen) {
						this.unseen_items.push(item)
					}
				}
				this.createFightPacks()
				if (store.state.farmer) { this.setFightPackPrice(store.state.farmer) }
				else { this.$root.$on('connected', (farmer: Farmer) => this.setFightPackPrice(farmer)) }

				if (this.unseen_items.length) {
					this.unseenItem = this.unseen_items[0]
					this.unseenItemDialog = true
				}

				this.update()
			})
			this.$root.$on('back', this.back)
			LeekWars.setActions(this.actions)
		}
		back() {
			this.$router.back()
		}
		destroyed() {
			this.$root.$off('back', this.back)
		}

		@Watch('$route.params.item')
		update() {
			const item = this.$route.params.item
			if (item) {
				this.selectedItem = this.items_by_name[item]
				LeekWars.setTitle(this.translateName(this.selectedItem))
				LeekWars.splitShowContent()
				this.$root.$emit('loaded')
			} else if (!LeekWars.mobile) {
				this.$router.replace('/market/pistol')
			} else {
				this.selectedItem = null
				LeekWars.setTitle(this.$t('title'))
				LeekWars.splitShowList()
				this.$root.$emit('loaded')
			}
			this.updateSubtitle()
			LeekWars.setActions(this.actions)
		}
		updateSubtitle() {
			if (this.$store.state.farmer) {
				LeekWars.setSubTitle(this.$t('main.x_habs', [LeekWars.formatNumber(this.$store.state.farmer.habs)]) + " • " + this.$t('main.x_crystals', [LeekWars.formatNumber(this.$store.state.farmer.crystals)]))
			}
		}
		translateName(item: ItemTemplate) {
			if (item.type === ItemType.FIGHT_PACK) {
				return this.$t('n_fights', [item.id - 1000000])
			}
			const type = ['weapon', 'chip', 'potion', 'hat', 'pomp'][item.type - 1]
			return this.$t(type + '.' + item.name.replace(type + '_', ''))
		}
		openBuyHabs() {
			if (this.selectedItem && this.selectedItem.price! <= this.$store.state.farmer.habs) {
				this.buyDialog = true
			}
		}
		openBuyCrystals() {
			if (this.selectedItem && this.selectedItem.crystals! <= this.$store.state.farmer.crystals) {
				this.buyCrystalsDialog = true
			}
		}
		buy(currency: string) {
			if (!this.selectedItem) { return }
			const item = this.selectedItem
			const method = currency === 'habs' ? 'market/buy-habs' : 'market/buy-crystals'
			const id = item.type === ItemType.FIGHT_PACK ? (item.id - 1000000) + 'fights' : item.id
			LeekWars.post(method, {item_id: id}).then(data => {
				this.buyDialog = false
				this.buyCrystalsDialog = false
				// _.toast([
				// 	_.lang.get('market', 'weapon_bought'),
				// 	_.lang.get('market', 'chip_bought'),
				// 	_.lang.get('market', 'potion_bought'),
				// 	_.lang.get('market', 'hat_bought'),
				// 	_.lang.get('market', 'fights_bought')
				// ][type - 1])
				if (item.type !== ItemType.FIGHT_PACK) {
					item.farmer_count!++
				}
				if (currency === 'habs') {
					this.$store.commit('update-habs', -item.price!)
				} else {
					this.$store.commit('update-crystals', -item.crystals!)
				}
				if (item.type === ItemType.FIGHT_PACK) {
					this.$store.commit('update-fights', data.fights)
				}
				this.$store.commit('add-inventory', {type: item.type, item_id: data.item, item_template: id})
				this.updateSubtitle()
			})
			.error(error => LeekWars.toast(this.$t('error_' + error.error, error.params)))
		}
		sell() {
			if (!this.selectedItem) { return }
			const item = this.selectedItem
			LeekWars.post('market/sell-habs', {item_id: item.id}).then(data => {
				this.sellDialog = false
				// _.toast([
				// 	_.lang.get('market', 'weapon_selled'),
				// 	_.lang.get('market', 'chip_selled'),
				// 	_.lang.get('market', 'potion_selled'),
				// 	_.lang.get('market', 'hat_selled')
				// ][type - 1])
				item.farmer_count!--
				this.$store.commit('update-habs', item.sell_price)
				this.$store.commit('remove-inventory', {type: item.type, item_template: item.id})
				this.updateSubtitle()
			})
			.error(error => LeekWars.toast(this.$t('error_' + error.error, error.params)))
		}
		updateChipMode() {
			this.chipMode = this.chipMode === 'level' ? 'type' : 'level'
			localStorage.setItem('market/sort_mode', this.chipMode)
		}
		createFightPacks() {
			const fights = [100, 200, 500, 1000]
			const costs = [1, 1.8, 4, 7]
			for (const p in fights) {
				const count = fights[p]
				const pack: ItemTemplate = {
					id: 1000000 + fights[p],
					name: count + '-fights',
					title: this.$t('n_fights', [count]),
					price: p === '0' ? 100000 : 0,
					crystals: costs[p] * 100,
					buyable: p === '0',
					buyable_crystals: true,
					sellable: false,
					type: ItemType.FIGHT_PACK,
					description: this.$t('n_fights_desc', [count]),
					leeks: [],
					leek_objs: [],
					leek_count: 0,
					farmer_count: 0,
					sell_price: 0,
					fights: count,
					seen: true,
					singleton: false,
					trophy: null,
					market: true,
					level: 1,
					params: null,
					public: true
				} as ItemTemplate
				this.fight_packs.push(pack)
				this.items[pack.id] = pack
				this.items_by_name[pack.name] = pack
			}
		}
		setFightPackPrice(farmer: Farmer) {
			const ratio = store.state.farmer!.total_level / 1204
			const priceHabs = Math.round(100000 + Math.pow(ratio, 3) * 4900000)
			this.items_by_name['100-fights'].price! = priceHabs
		}

		@Watch('unseenItemDialog')
		updateUnseenDialog() {
			if (!this.unseenItemDialog) {
				LeekWars.post('market/item-seen', {item: this.unseenItem!.id})
				setTimeout(() => {
					this.unseen_items.shift()
					if (this.unseen_items.length) {
						this.unseenItem = this.unseen_items[0]
						this.unseenItemDialog = true
					}
				}, 200)
			}
		}

		scroll(index: number) {
			const position = document.querySelectorAll('.column8 h2')[index].getBoundingClientRect().top + window.scrollY
			window.scrollTo(0, position)
		}
	}
</script>

<style lang="scss" scoped>
	.preview-panel .loader {
		padding: 80px 0;
	}
	.column4 {
		position: sticky;
		top: 12px;
		max-height: calc(100vh - 24px);
		display: inline-flex;
		flex-direction: column;
		& > * {
			min-height: 0;
		}
	}
	.preview.content {
		text-align: center;
		padding: 0;
	}
	.preview {
		overflow-y: auto;
	}
	.preview .leeks {
		padding: 6px 0;
	}
	.preview .leek {
		background: #5fad1b;
		color: white;
		padding: 4px 8px;
		border-radius: 3px;
		margin: 3px 1px;
		display: inline-block;
	}
	.items {
		display: grid;
		grid-gap: 8px;
		padding: 8px;
	}
	.items.potions {
		grid-template-columns: repeat(auto-fill, minmax(80px, 1fr));
	}
	.items.chips {
		grid-template-columns: repeat(auto-fill, minmax(76px, 1fr));
	}
	.items.weapons {
		grid-template-columns: repeat(auto-fill, minmax(165px, 1fr));
	}
	.items.fights {
		grid-template-columns: repeat(auto-fill, minmax(160px, 1fr));
	}
	.items.hats {
		grid-template-columns: repeat(auto-fill, minmax(100px, 1fr));
	}
	.items.pomps {
		grid-template-columns: repeat(auto-fill, minmax(100px, 1fr));
	}
	.items .item {
		border: 1px solid #ddd;
		cursor: pointer;
		position: relative;
		text-align: center;
		&.toohigh {
			opacity: 0.4;
		}
	}
	.items .item.router-link-active {
		background: white;
		box-shadow: 0px 2px 1px -1px rgba(0,0,0,0.2), 0px 1px 1px 0px rgba(0,0,0,0.14), 0px 1px 3px 0px rgba(0,0,0,0.12);
	}
	.buy-label {
		display: inline-block;
	}
	.items .item:not([leek-count="0"]):before {
		position: absolute;
		bottom: -5px;
		right: -5px;
		min-width: 20px;
		height: 20px;
		padding: 1px 5px;
		content: attr(leek-count);
		text-align: center;
		color: #eee;
		border-radius: 20px;
		background-color: #5fad1b;
		font-weight: bold;
		z-index: 1;
	}
	.items .item:not([farmer-count="0"]):after {
		position: absolute;
		bottom: -5px;
		right: -5px;
		min-width: 20px;
		height: 20px;
		padding: 1px 5px;
		content: attr(farmer-count);
		text-align: center;
		color: #eee;
		border-radius: 20px;
		font-weight: bold;
		background-color: #777;
	}
	.items .item:not([farmer-count="0"]):not([leek-count="0"]):before {
		border-radius: 20px 0 0 20px;
		right: 16px;
	}
	.items .item:not([farmer-count="0"]):not([leek-count="0"]):after {
		border-radius: 0 20px 20px 0;
	}
	.items .item.too-expensive img {
		opacity: 0.4;
	}
	.fight-pack {
		padding: 10px;
		font-size: 16px;
		color: #777;
	}
	.fights img {
		height: 75px;
		margin-bottom: -3px;
	}
	.weapons .weapon {
		padding: 10px;
		height: 70px;
		img {
			max-height: 60px;
			max-width: 150px;
			position: absolute;
			top: 0;
			bottom: 0;
			left: 0;
			right: 0;
			margin: auto;
		}
	}
	.chips .chip {
		padding: 6px;
	}
	.chips .chip img {
		width: 62px;
		vertical-align: bottom;
	}
	.panel h4 {
		margin: 0;
		margin-left: 8px;
		margin-bottom: 0;
		font-size: 15px;
		&.first {
			margin-top: 8px;
		}
	}
	.potion {
		padding: 6px 0;
	}
	.potion img {
		width: 80px;
	}
	.hat {
		height: 82px;
		padding: 6px;
	}
	.hat img {
		max-width: 92px;
		max-height: 70px;
		position: absolute;
		top: 0;
		bottom: 0;
		left: 0;
		right: 0;
		margin: auto;
	}
	.buy-buttons {
		padding-top: 10px;
		padding-bottom: 10px;
		button {
			margin-bottom: 5px;
			margin-left: 8px;
			img {
				vertical-align: top;
				margin-left: 5px;
			}
		}
	}
	.buy-crystals-button img {
		vertical-align: middle;
		margin-top: -15px;
		margin-bottom: -15px;
		margin-left: 5px;
		height: 40px;
	}
	.trophy {
		padding: 6px;
		display: flex;
		align-items: center;
		justify-content: center;
		img {
			width: 20px;
			margin-right: 4px;
		}
	}
	.already-have {
		font-style: italic;
		color: #777;
		padding-bottom: 8px;
	}
	.unseen-dialog .trophy {
		margin: 10px;
		padding: 10px;
	}
	.pomps {
		.pomp {
			padding: 6px;
			img {
				vertical-align: bottom;
				width: 75px;
			}
		}
	}
	.menu {
		display: grid;
		grid-template-columns: repeat(auto-fill, minmax(100px, 1fr));
		grid-gap: 8px;
		user-select: none;
		flex: 0 0 auto;
		.item {
			background: #f2f2f2;
			border-radius: 4px;
			box-shadow: 0px 10px 11px -11px rgba(0,0,0,0.75);
			padding: 8px 5px;
			cursor: pointer;
			color: #555;
			&:hover {
				background: white;
				color: black;
			}
		}
	}
</style>
