<template>
    <modal
        class="modal min-w-modal"
        tabindex="-1"
        role="dialog"
        @modal-close="handleClose"
    >
        <loading-view :loading="loading">
            <card class="overflow-hidden  max-w-modal min-w-modal">
                <form
                    v-if="panels"
                    @submit="submitViaUpdateResource"
                    autocomplete="off"
                    ref="form"
                >

                    <div v-for="panel in panelsWithFields" :key="panel.name">
                        <card>
                            <component

                                :class="{
                                'remove-bottom-border': index == panel.fields.length - 1,
                            }"
                                v-for="(field, index) in panel.fields"
                                :key="index"
                                :is="`form-${field.component}`"
                                :errors="validationErrors"
                                :resource-id="resourceId"
                                :resource-name="resourceName"
                                :field="field"
                                :via-resource="viaResource"
                                :via-resource-id="viaResourceId"
                                :via-relationship="viaRelationship"
                                @file-deleted="$emit('update-last-retrieved-at-timestamp')"
                            />
                        </card>
                    </div>


                    <div class="bg-30 flex px-8 py-4">
                        <button
                            type="button"
                            @click="handleClose"
                            class="ml-auto btn btn-default btn-danger mr-3"
                        >
                            {{__('Cancel')}}
                        </button>
                        <progress-button
                            dusk="update-button"
                            type="submit"
                            :disabled="isWorking"
                            :processing="wasSubmittedViaUpdateResource"
                        >
                            {{ __('Update :resource', { resource: singularName }) }}
                        </progress-button>
                    </div>
                </form>
            </card>
        </loading-view>
    </modal>
</template>

<script>
    import {Errors, InteractsWithResourceInformation} from 'laravel-nova'


    export default {
        mixins: [InteractsWithResourceInformation],

        props: {
            resourceId: {
                required: true
            },
            resourceName: {
                required: true
            },

        },

        data: () => ({
            relationResponse: null,
            loading: true,
            submittedViaUpdateResource: false,
            fields: [],
            panels: [],
            validationErrors: new Errors(),
            lastRetrievedAt: null,
            isWorking: false,
            savedItem: false,
        }),
        watch: {},
        async created() {
            if (Nova.missingResource(this.resourceName))
                return this.$router.push({name: '404'})

            // If this update is via a relation index, then let's grab the field
            // and use the label for that as the one we use for the title and buttons
            if (this.isRelation) {
                const {data} = await Nova.request(
                    `/nova-api/${this.viaResource}/field/${this.viaRelationship}`
                )
                this.relationResponse = data
            }

            this.getFields()
            this.updateLastRetrievedAtTimestamp()
        },
        mounted() {

        },
        methods: {
            handleClose() {
                this.$emit(this.savedItem ? 'confirm' : 'close')
            },

            /**
             * Get the available fields for the resource.
             */
            async getFields() {
                this.loading = true

                this.panels = []
                this.fields = []

                const {
                    data: {panels, fields},
                } = await Nova.request()
                    .get(
                        `/nova-api/${this.resourceName}/${this.resourceId}/update-fields`,
                        {
                            params: {
                                editing: true,
                                editMode: 'update',
                                viaResource: this.viaResource,
                                viaResourceId: this.viaResourceId,
                                viaRelationship: this.viaRelationship,
                            },
                        }
                    )
                    .catch(error => {
                        if (error.response.status == 404) {
                            this.$router.push({name: '404'})
                            return
                        }
                    })

                this.panels = panels
                this.fields = fields
                this.loading = false

                Nova.$emit('resource-loaded')
            },

            async submitViaUpdateResource(e) {
                e.preventDefault()
                this.submittedViaUpdateResource = true
                await this.updateResource()
            },

            async submitViaUpdateResourceAndContinueEditing() {
                this.submittedViaUpdateResource = false
                await this.updateResource()
            },

            /**
             * Update the resource using the provided data.
             */
            async updateResource() {
                this.isWorking = true

                if (this.$refs.form.reportValidity()) {
                    try {
                        const {
                            data: {redirect},
                        } = await this.updateRequest()

                        Nova.success(
                            this.__('The :resource was updated!', {
                                resource: this.resourceInformation.singularLabel.toLowerCase(),
                            })
                        )

                        await this.updateLastRetrievedAtTimestamp()
                        this.savedItem = true;
                        this.handleClose();
                        return;
                    } catch (error) {
                        this.submittedViaUpdateResource = false

                        if (error.response.status == 422) {
                            this.validationErrors = new Errors(error.response.data.errors)
                            Nova.error(this.__('There was a problem submitting the form.'))
                        }

                        if (error.response.status == 409) {
                            Nova.error(
                                this.__(
                                    'Another user has updated this resource since this page was loaded. Please refresh the page and try again.'
                                )
                            )
                        }
                    }
                }

                this.submittedViaUpdateResource = false
                this.isWorking = false
            },

            /**
             * Send an update request for this resource
             */
            updateRequest() {
                return Nova.request().post(
                    `/nova-api/${this.resourceName}/${this.resourceId}`,
                    this.updateResourceFormData,
                    {
                        params: {
                            viaResource: this.viaResource,
                            viaResourceId: this.viaResourceId,
                            viaRelationship: this.viaRelationship,
                            editing: true,
                            editMode: 'update',
                        },
                    }
                )
            },

            /**
             * Update the last retrieved at timestamp to the current UNIX timestamp.
             */
            updateLastRetrievedAtTimestamp() {
                this.lastRetrievedAt = Math.floor(new Date().getTime() / 1000)
            },
        },

        computed: {

            wasSubmittedViaUpdateResource() {
                return this.isWorking && this.submittedViaUpdateResource
            },

            /**
             * Create the form data for creating the resource.
             */
            updateResourceFormData() {
                return _.tap(new FormData(), formData => {
                    _(this.fields).each(field => {
                        field.fill(formData)
                    })

                    formData.append('_method', 'PUT')
                    formData.append('_retrieved_at', this.lastRetrievedAt)
                })
            },

            singularName() {
                if (this.relationResponse) {
                    return this.relationResponse.singularLabel
                }

                return this.resourceInformation.singularLabel
            },

            isRelation() {
                return Boolean(this.viaResourceId && this.viaRelationship)
            },

            panelsWithFields() {
                return _.map(this.panels, panel => {
                    return {
                        name: panel.name,
                        fields: _.filter(this.fields, field => field.panel == panel.name),
                    }
                })
            },

            /**
             * These are here to satisfy the parameter requirements for deleting the resource
             */
            currentSearch() {
                return ''
            },

            encodedFilters() {
                return []
            },

            currentTrashed() {
                return ''
            },

            viaResource() {
                return ''
            },

            viaResourceId() {
                return ''
            },

            viaRelationship() {
                return ''
            },

            selectedResources() {
                return [this.resourceId]
            },


        }
    }
</script>
