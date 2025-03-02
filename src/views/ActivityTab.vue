<!--
  - @copyright Copyright (c) 2021 Louis Chemineau <louis@chmn.me>
  -
  - @author Louis Chemineau <louis@chmn.me>
  -
  - @license GNU AGPL version 3 or any later version
  -
  - This program is free software: you can redistribute it and/or modify
  - it under the terms of the GNU Affero General Public License as
  - published by the Free Software Foundation, either version 3 of the
  - License, or (at your option) any later version.
  -
  - This program is distributed in the hope that it will be useful,
  - but WITHOUT ANY WARRANTY; without even the implied warranty of
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program. If not, see <http://www.gnu.org/licenses/>.
  -
  -->

<template>
	<div :class="{ 'icon-loading': loading }">
		<!-- error message -->
		<EmptyContent v-if="error" icon="icon-error">
			{{ error }}
		</EmptyContent>
		<template v-else>
			<!-- activities content -->
			<ul>
				<Activity
					v-for="activity in activities"
					:key="activity.id"
					:activity="activity" />
			</ul>

			<EmptyContent v-if="activities.length === 0 && !loading" icon="icon-activity">
				{{ t('activity', 'No activity yet') }}
			</EmptyContent>
		</template>
	</div>
</template>

<script>
import { generateOcsUrl } from '@nextcloud/router'
import axios from '@nextcloud/axios'
import EmptyContent from '@nextcloud/vue/dist/Components/EmptyContent'

import Activity from '../components/Activity'
import ActivityModel from '../models/ActivityModel'

import logger from '../logger'

export default {
	name: 'ActivityTab',
	components: {
		Activity,
		EmptyContent,
	},
	data() {
		return {
			error: '',
			loading: true,
			fileInfo: null,
			activities: [],
		}
	},
	methods: {
		/**
		 * Update current fileInfo and fetch new activities
		 * @param {Object} fileInfo the current file FileInfo
		 */
		async update(fileInfo) {
			this.fileInfo = fileInfo
			this.resetState()
			await this.getActivities()
		},
		/**
		 * Get the existing activities
		 */
		async getActivities() {
			try {
				this.loading = true

				const activities = await axios.get(
					generateOcsUrl('apps/activity/api/v2/activity/filter'),
					{
						params: {
							format: 'json',
							object_type: 'files',
							object_id: this.fileInfo.id,
						},
					})

				this.loading = false

				this.processActivities(activities)
			} catch (error) {
				// Status 304 is not an error.
				if (error.response !== undefined && error.response.status === 304) {
					this.loading = false
					return
				}
				this.error = t('activity', 'Unable to load the activity list')
				this.loading = false
				console.error('Error loading the activity list', error)
			}
		},
		/**
		 * Reset the current view to its default state
		 */
		resetState() {
			this.loading = true
			this.error = ''
			this.activities = []
		},
		/**
		 * Process the current activity data
		 * and init activities[]
		 *
		 * @param {Object} activity the activity ocs api request data
		 * @param {Object} activity.data the request data
		 */
		processActivities({ data }) {
			if (data.ocs && data.ocs.data && data.ocs.data.length > 0) {
				// create Activity objects and sort by newest
				this.activities = data.ocs.data
					.map(activity => new ActivityModel(activity))
					.sort((a, b) => b.timestamp - a.timestamp)

				logger.debug(`Processed ${this.activities.length} activity(ies)`, { activities: this.activities, fileInfo: this.fileInfo })
			}
		},
	},
}
</script>
