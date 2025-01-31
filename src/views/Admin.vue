<template>
	<top-nav />

	<side-nav />

	<!--	Admin Results	-->
    <v-main v-if="$store.getters['auth/getUser'] !== null">
        <!-- results -->
		<v-table 
			v-if="$route.params.eventSlug && event" 
			density="comfortable" :bordered="true" 
			hover
			:height="scoreSheetHeight"
			fixed-header
		>
			<thead>
				<tr>
					<th colspan="2" class="text-center text-uppercase font-weight-bold">{{ event.title}}</th>
					<template v-for="(technical, technicalKey, technicalIndex) in technicals" :key="technical.id">
						<th class="text-center text-uppercase font-weight-bold text-red-darken-3">
							Deduct {{ technicalIndex + 1 }}
						</th>
					</template>
					<template v-for="judge in judges" :key="judge.id">
						<th colspan="2" class="text-center text-uppercase font-weight-bold">Judge {{ judge.number }}</th>
					</template>
					<th class="text-center text-uppercase font-weight-bold text-green-darken-4">Average</th>
					<th class="text-center text-uppercase font-weight-bold text-blue-darken-4">Total Rank</th>
					<th class="text-center text-uppercase font-weight-bold text-grey-darken-1">Initial Rank</th>
					<th class="text-center text-uppercase font-weight-bold">Final Rank</th>
				</tr>
			</thead>
			
			<tbody>
			<tr v-for="(team, teamKey, teamIndex) in teams" :key="team.id">
				<td class="text-h5 text-center font-weight-bold">{{ teamIndex + 1 }}</td>
				<td class="text-center text-uppercase font-weight-bold" :style="{'color': team.color}">{{ team.name }}</td>
				 <template v-for="(technical, technicalKey, technicalIndex) in technicals" :key="technical.id">
					<td
						class="text-center text-uppercase font-weight-bold text-red-darken-3"
						:class="{
							'bg-grey-lighten-3' : !team.deductions.inputs[technicalKey].is_locked,
							'bg-white' : team.deductions.inputs[technicalKey].is_locked
						}"
					>
						{{ team.deductions.inputs[technicalKey].value.toFixed(2) }}
					</td>
				</template> 
				<template v-for="judge in judges" :key="judge.id">
					<td
						class="text-center text-red-darken-3"
						:class="{
							'bg-grey-lighten-3' : !team.ratings.inputs[`judge_${judge.id}`].final.is_locked,
							'bg-white' : team.ratings.inputs[`judge_${judge.id}`].final.is_locked
						}"
					>
						{{ team.ratings.inputs[`judge_${judge.id}`].final.deducted.toFixed(2) }}
					</td>
					<td
						class="text-center font-weight-bold text-blue-darken-2"
						:class="{
							'bg-grey-lighten-3' : !team.ratings.inputs[`judge_${judge.id}`].final.is_locked,
							'bg-white' : team.ratings.inputs[`judge_${judge.id}`].final.is_locked
						}"
					>
						{{ team.ratings.inputs[`judge_${judge.id}`].rank.fractional.toFixed(2) }}
					</td>
				</template>
				<td class="text-center font-weight-bold text-green-darken-4">{{ team.ratings.average.toFixed(2) }}</td>
				<td class="text-center font-weight-bold text-blue-darken-4">{{ team.rank.total.fractional.toFixed(2) }}</td>
				<td class="text-center font-weight-bold text-grey-darken-1">{{ team.rank.initial.fractional.toFixed(2) }}</td>
				<td class="text-center font-weight-bold">{{ team.rank.final.fractional }}</td>
			</tr>
			</tbody>
			<tfoot>
				<tr>
					<td colspan="20">
						<v-row>
							<template v-for="technical in technicals" :key="technical.id">
								<v-col>
									<v-card class="text-center mb-5" flat>
										<v-card-title class="pt-16 font-weight-bold">
											{{ technical.name }}
										</v-card-title>
										<v-card-text class="text-center">
											Technical Judge {{ technical.number }}
										</v-card-text>
									</v-card>
								</v-col>
							</template>

							<template v-for="judge in judges" :key="judge.id">
								<v-col>
									<v-card class="text-center mb-5" flat>
										<v-card-title class="pt-16 font-weight-bold">
											{{ judge.name }}
										</v-card-title>
										<v-card-text class="text-center">
											Judge {{ judge.number }} <template v-if="judge.is_chairman == 1">(Chairman)</template>
										</v-card-text>
									</v-card>
								</v-col>
							</template>
						</v-row>
					</td>
				</tr>
			</tfoot>
		</v-table>

        <!-- loader -->
        <div v-else-if="this.$route.params.eventSlug" class="d-flex justify-center align-center" style="height: 100vh;">
            <v-progress-circular
                :size="80"
                color="black"
                class="mb-16"
                indeterminate
            />
        </div>
    </v-main>
</template>
<script>
	import TopNav from "../components/nav/TopNav.vue";
	import SideNav from "../components/nav/SideNav.vue";
    import $ from 'jquery';

    export default {
        name: 'Admin',
		components: {
            TopNav,
            SideNav
		},
        data() {
            return {
                event: null,
                results: {},
                timer: null,
				teamIndex: 0,
				teams: [],
				judges: [],
				technicals: []
            }
        },
		computed: {
			scoreSheetHeight() {
				return this.$store.getters.windowHeight - 64;
			}
		},
        watch: {
            $route: {
                immediate: true,
                handler(to, from) {
                    this.event = null;
                    if(this.timer)
                        clearTimeout(this.timer);
                    this.tabulate();
                }
            }
        },
        methods: {
            async tabulate() {
                // tabulate selected event
                if (this.$route.params.eventSlug) {
                    await $.ajax({
                        url: `${this.$store.getters.appURL}/admin.php`,
                        type: 'GET',
                        xhrFields: {
                            withCredentials: true
                        },
                        data: {
                            tabulate: this.$route.params.eventSlug
                        },
                        success: (data) => {
                            data = JSON.parse(data);
                            this.event = data.event;
							this.teams = data.results.teams;
							this.judges = data.results.judges;
							this.technicals = data.results.technicals;
							console.log(data)
                            // request again
                            if(data.event.slug === this.$route.params.eventSlug) {
                                this.timer = setTimeout(() => {
                                    this.tabulate();
                                }, 2400);
                            }
                        },
                        error: (error) => {
                            alert(`ERROR ${error.status}: ${error.statusText}`);
                        },
                    });
                }
            }
		}
    }
</script>

<style scoped>
tbody td, th {
	height: 64px !important;
}
tbody td {
	border-bottom: 1px solid #ddd;
	padding: 1rem !important;
}
th, td {
	border: 1px solid #ddd;
}
</style>
