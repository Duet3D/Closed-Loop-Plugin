<template>
	<v-card class="fill-height">
		<v-card-title class="pt-2 pb-1">
			<v-icon class="mr-2">mdi-record-rec</v-icon>
			Record
		</v-card-title>

		<v-card-text>
			<v-row>
				<v-col v-for="colNo in [0,1,2]" :key="colNo" cols="2">
					<v-checkbox
						dense
						hide-details
						v-for="variable in nthThirdOfVariables(colNo)"
						:key="variable.id"
						v-model="selectedVariables"
						:label="variable.title"
						:value="variable"
						:color="variable.colour"
					/>
				</v-col>
				<v-col cols="3">
					<v-select
						v-model="selectedDriver"
						:items="drivers"
						hint="Axes with > 1 drive will not show."
						item-text="name"
						item-value="value"
						label="Select a driver"
						single-line
						persistent-hint
						class="pb-4"
					/>
					<v-row>
						<v-col cols="6">
							<v-text-field
								label="Samples to collect"
								v-model="sampleCount"
								:rules="[v => !!v || $t('dialog.inputRequired'), v => isNumber(parseFloat(v)) || $t('dialog.numberRequired')]"
								required
								autofocus
							>
								<template #append>
									samples
								</template>
							</v-text-field>
						</v-col>
						<v-col cols="6">
							<v-radio-group v-model="sampleRateContinuous">
								<v-radio
									label="As fast as possible"
									:value="true"
									dense
									hide-details
								/>
								<v-radio
									:value="false"
									dense
									hide-details
								>
									<template v-slot:label>
										<v-text-field
											label="At a fixed rate"
											v-model="sampleRate"
											:rules="[v => !!v || $t('dialog.inputRequired'), v => isNumber(parseFloat(v)) || $t('dialog.numberRequired')]"
											required
											autofocus
											:hint="sampleRateContinuous ? '' : `(total ${totalTime} seconds)`"
											persistent-hint
										>
											<template slot="append">
												/second
											</template>
										</v-text-field>
									</template>
								</v-radio>
							</v-radio-group>
						</v-col>
					</v-row>
					<v-select
						:items="[{text: 'Immediately', value: 0}, {text: 'On next move', value: 1}]"
						label="Collect data"
						v-model="activateMode"
						:disabled="calibrationMovement > 0"
						:hint="calibrationMovement > 0 ? 'Only immediate collection is supported when a tuning move is selected.' : ''"
						persistent-hint
					></v-select>
				</v-col>
				<v-col cols="3">
					<v-radio-group class="mt-0 pt-0" label="Movement" v-model="calibrationMovement">
						<v-radio
							label="None"
							:value="-1"
							dense
							hide-details
						/>
						<v-radio
							v-for="manoeuvre in tuningManoeuvres"
							:key="manoeuvre.value"
							:label="(manoeuvre.disabled ? '[Coming Soon!] ' : '') + manoeuvre.name"
							:value="manoeuvre.value"
							:disabled="manoeuvre.disabled"
							dense
							hide-details
						/>
						<v-radio
							:value="0"
							hide-details
						>
							<template v-slot:label>
								<v-text-field
									label="Custom GCODE"
									v-model="customGCODE"
								/>
							</template>
						</v-radio>
					</v-radio-group>
				</v-col>
			</v-row>
			<v-row>
				<v-col cols="auto">
					<v-btn :disabled="!ready" @click="record()" color="info">
						<v-icon class="mr-2">mdi-record</v-icon>
						Record
					</v-btn>
				</v-col>
				<v-col cols="10">
					<div v-if="ready" :class="{'pt-2': !error && !warning && recordingProgress == null && calibrationMovement != 0}" class="font-weight-black info--text">{{ GCODECommand }}</div>
					<div v-if="ready && calibrationMovement == 0" class="font-weight-black info--text">{{ customGCODE }}</div>
					<div v-if="error" class="font-weight-black error--text">{{ error }}</div>
					<div v-if="warning" class="font-weight-black warning--text">{{ warning }}</div>
					<v-progress-linear
						v-if="recordingProgress != null"
						:indeterminate="recordingProgress == -1"
						:value="recordingProgress == -1 ? 0 : recordingProgress"
						class="mb-4"
					/>
				</v-col>
			</v-row>
		</v-card-text>

		<v-dialog v-model="showDialog" persistent width="480">
			<v-card>
				<v-card-title>
					<span class="headline">
						Confirm
					</span>
				</v-card-title>

				<v-card-text>
					You have chosen to perform a tuning manoeuvre which will move the axis a small distance.
					<div class="text-center py-2 font-weight-black error--text">
						This movement may not respect endstops.
					</div>
					Please ensure the axis is in a safe position (usually the center) before proceeding.
				</v-card-text>

				<v-card-actions>
					<v-btn color="error darken-1" text @click="dialogResult(false)">
						{{ $t('generic.cancel') }}
					</v-btn>
					<v-spacer/>
					<v-checkbox v-model="dontShowModal" class="pr-2">
						<template v-slot:label>
							<span style="font-size: 0.7em">Don't show this again</span>
						</template>
					</v-checkbox>
					<v-btn color="blue darken-1" text @click="dialogResult(true)">
						{{ $t('generic.ok') }}
					</v-btn>
				</v-card-actions>
			</v-card>
		</v-dialog>
	</v-card>
</template>

<script>
'use strict'

import { mapState, mapActions } from 'vuex'

import { variables, tuningManoeuvres } from './config.js'

// import { mapState, mapGetters, mapActions } from 'vuex'

export default {
	data: () => ({
		variables,
		error: null,
		warning: null,
		sampleRate: 100,
		activateMode: 0,
		tuningManoeuvres,
		sampleCount: 500,
		customGCODE: null,
		showDialog: false,
		selectedDriver: null,
		dontShowModal: false,
		selectedVariables: [],
		calibrationMovement: -1,
		recordingProgress: null,      // null = not recording, -1 = recording but unknown progress, >=0 = recording & known progress
		sampleRateContinuous: true,
	}),
	methods: {
		...mapActions('machine', ['sendCode']),
		nthThirdOfVariables(n) {
			let thirdLength = Math.ceil(this.variables.length / 3);
			return this.variables.slice(n * thirdLength, (n+1) * thirdLength);
		},
		dialogResult(res) {
			this.showDialog = false;
			if (res) {
				this.record(true);
			}
		},
		async record(force = false) {
			force |= this.dontShowModal;
			if (this.calibrationMovement !== -1 && !force) {
				this.showDialog = true;
				return;
			}

			const gcodeToSend = (this.calibrationMovement === 0) ? (this.GCODECommand + '\n' + this.customGCODE) : this.GCODECommand;
			const reply = await this.sendCode({ code: gcodeToSend, fromInput: false });
			if (reply.startsWith('Error: ')) {
				this.error = reply;
				return;
			} else if (reply.startsWith('Warning: ')) {
				this.warning = reply;
			}

			this.error = null;
			this.warning = null;
			this.recordingProgress = -1;
		}
	},
	computed: {
		...mapState('machine/model', {
			boards: state => state.boards,
			axes: state => state.move.axes
		}),
		drivers() {
			return this.axes
				.filter(axis => axis && axis.drivers.length === 1 && axis.drivers[0])
				.filter(axis => this.boards.some(board => board && board.canAddress === parseInt(axis.drivers[0].split('.')[0]) && board.closedLoop !== null))
				.map(axis => ({
					name: `${axis.letter} axis (driver ${axis.drivers[0]})`,
					value: axis.drivers[0]
				}));
		},
		closedLoopPoints() {
			if (this.selectedDriver) {
				const canAddress = parseInt(this.selectedDriver.split('.')[0]);
				const board = this.boards.find(board => board.canAddress === canAddress);
				if (board && board.closedLoop) {
					return board.closedLoop.points;
				}
			}
			return null;
		},
		closedLoopRuns() {
			if (this.selectedDriver) {
				const canAddress = parseInt(this.selectedDriver.split('.')[0]);
				const board = this.boards.find(board => board.canAddress === canAddress);
				if (board && board.closedLoop) {
					return board.closedLoop.runs;
				}
			}
			return null;
		},
		totalTime() {
			return Math.round((this.sampleCount / this.sampleRate) * 100) / 100;
		},
		GCODECommand() {
			const pString = `P${this.selectedDriver}`;
			const sString = `S${this.sampleCount}`;
			const aString = `A${this.activateMode}`;
			const rString = `R${this.sampleRateContinuous ? 0 : this.sampleRate}`;
			const dString = `D${this.selectedVariables.reduce((acc, x) => acc + x.filterValue, 0)}`;
			const vString = `V${this.calibrationMovement > 0 ? this.calibrationMovement : 0}`;
			return `M569.5 ${pString} ${sString} ${aString} ${rString} ${dString} ${vString}`;
		},
		ready() {
			return this.selectedDriver !== null && this.selectedVariables.length > 0;
		},
	},
	watch: {
		calibrationMovement() {
			this.activateMode = 0;
		},
		closedLoopPoints(to) {
			if (this.recordingProgress !== null) {
				this.recordingProgress = (to / this.sampleCount) * 100;
			}
		},
		closedLoopRuns(to) {
			if (this.recordingProgress !== null) {
				this.recordingProgress = null;
				this.$emit("recordingFinished");
			}
		}
	}
}
</script>