<template>
    <div>
        <slot>
            <div class="ui very padded container">
                <img class="ui centered small circular image" v-if="imgSrc" :src="src">
                <semantic-divider />
                <a class="ui fluid blue button" @click="showModal = !showModal">{{ buttonText }}</a>  
            </div>
        </slot>

        <semantic-modal size="small" :title="title" :active="showModal" :settings="modalSetting">
            <div class="ui basic segment">
                <div class="Image-upload-wrapper Image-upload">
                    
                    <div ref="croppie"></div>
                    <div id="upload-wrapper" class="upload-wrapper">
                        <div class="input-file">
                            <input ref="select" name="image-upload" type="file" accept="image/*" id="upload-image" v-on:change="setUpFileUploader" style="display: none">
                        </div>

                        <button class="ui blue button" @click="triggerSelect">
                            Select File
                        </button>

                        <button class="circular ui primary icon button" @click="croppie.rotate(-90)">
                        <i class="black undo icon"></i>
                        </button>

                        <button class="circular ui primary icon button" @click="croppie.rotate(90)">
                        <i class="black repeat icon"></i>
                        </button>
                    </div>
                </div>
                
                <semantic-divider class="hidden" />
                <button class="ui right floated blue button" :class="{'loading': loading}" id="uploadFileCall" @click="uploadFile" :disabled="canSet">Set</button>
                <button class="ui right floated button" :disabled="loading" @click="showModal = !showModal">Cancel</button>

            </div>
        </semantic-modal>
    </div>
</template>

<script>
    import Croppie from 'croppie'

    /**
    * An image cropper and uploader useful for user profiles
    *
    * @example ./croud-image-uploader.md
    */

    export default {
        name: 'croud-image-uploader',

        props: {
            /**
            * A Title for the semantic modal which contains the image cropper / uploader
            */
            title: {
                type: String,
                default: 'Set Image',
            },

            /**
            * The text displayed in the profile action button
            */
            buttonText: {
                type: String,
                default: 'Set Image',
            },

            /**
            * Current picture url
            */
            imgUrl: {
                type: String,
                default: '/img/defaultAvatar.png',
            },

            /**
            * Fallback default picture / placeholder
            */
            defaultSrc: {
                type: String,
                default: '/img/defaultAvatar.png',
            },

            /**
            * Request part of the url if you wish to use the upload function, the response is emitted to the parent attached to a image-set event
            */
            url: {
                type: String,
                required: false,
            },

            /**
            * Default Croud request headers are set for files endpoint, pass requestHeaders object to override them
            */
            requestHeaders: {
                type: Object,
                default() {
                    return {
                        'content-Type': 'multipart/form-data',
                        Cdn: 'cdn',
                    }
                },
            },

            /**
            * Croppie Settings
            */
            croppieSettings: {
                type: Object,
                default() {
                    return {
                        viewport: { width: 220, height: 220, type: 'square' },
                        boundary: { width: 260, height: 260 },
                        showZoomer: true,
                        enableOrientation: true,
                        format: 'png',
                        enforceBoundary: false,
                    }
                },
            },
        },

        data() {
            return {
                src: this.defaultSrc,
                loading: false,
                showModal: false,
                croppie: null,
                image: null,
                eventListenerAdded: false,
                //
                modalSetting: {
                    closable: true,
                    closable_button: true,
                    onVisible: () => {
                        this.image = this.cors
                        this.$nextTick(() => {
                            this.croppie.bind({
                                url: this.image,
                            })
                        })
                    },
                },
            }
        },

        mounted() {
            this.setUpCroppie()
            //
            this.$on('imgUploaded', ((imageData) => {
                this.image = imageData
                this.croppie.destroy()
                this.setUpCroppie(imageData)
                this.$nextTick(() => {
                    this.croppie.bind({
                        url: this.image,
                    })
                })
            }))
        },

        methods: {
            triggerSelect() {
                this.$refs.select.click()
            },

            profileSet(image) {
                this.src = image
            },

            uploadFile() {
                this.loading = true

                this.croppie.result(
                    { type: 'blob',
                        size: 'viewport',
                        name: 'profile',
                    }).then((response) => {
                        this.image = response
                        // Axios request
                        if (this.url) {
                            const formData = new FormData()
                            formData.append('file', this.image)
                            this.$http.post(this.url, formData, {
                                headers: this.requestHeaders,
                            }).then((res) => {
                                this.$emit('image-set', res.data.data)
                                this.$emit('modal-closed')
                                this.loading = false
                            })
                        }
                        this.profileSet(window.URL.createObjectURL(response))
                        this.showModal = !this.showModal
                        this.loading = false
                    })
            },

            setUpCroppie() {
                const el = this.$refs.croppie
                this.croppie = new Croppie(el, this.croppieSettings)
            },

            setUpFileUploader(e) {
                const files = e.target.files || e.dataTransfer.files
                if (!files.length) {
                    return
                }
                this.createImage(files[0])
            },

            createImage(file) {
                const reader = new FileReader()
                const vm = this

                reader.onload = (e) => {
                    vm.image = e.target.result
                    vm.$emit('imgUploaded', e.target.result)
                }
                reader.readAsDataURL(file)
            },
        },

        computed: {
            cors() {
                return `${this.src}?v=cors` || ''
            },

            imgSrc() {
                if (this.imgUrl) {
                    const tester = new Image()

                    if (!this.eventListenersAdded) {
                        tester.addEventListener('load', (() => {
                            this.src = this.imgUrl
                        }))

                        tester.addEventListener('error', (() => {
                            this.src = this.defaultSrc
                        }))
                    }

                    this.eventListenerAdded = true

                    tester.src = this.imgUrl
                }
                return true
            },

            computedUrl() {
                if (this.url.match(/:\/\//)) return this.url
                return `//${gateway_url}/${this.url}`
            },

            canSet() {
                return this.image === `${this.defaultSrc}?v=cors`
            },
        },
    }
</script>

<style lang="scss">

@import '~croppie/croppie.css';

.upload-wrapper {
    text-align: center;
}

</style>