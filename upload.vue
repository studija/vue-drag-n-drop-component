<template>
    <div>
        <slot></slot>
        <div class="v-grag-n-drop-container" v-bind:id="prop_component_id" @dragenter="onDragEnter" @dragleave="onDragLeave"
             @dragover="onDragOver" @drop="onDrop">
            <ul style="color: red;">
                <li v-for="error in attachments.errors">
                    <span v-if="error.validation_type === 'mime'">
                        {{(translations[error.validation_type]).replaceAll([':fileName', ':fileType'], [error.name, error.type]) }}
                    </span>
                    <span v-if="error.validation_type === 'single_size'">
                        {{(translations[error.validation_type]).replaceAll([':fileName', ':fileMaxSize', ':fileSize'], [error.name, precisionRound(convertToMb(input.max_single_file_size), 2), precisionRound(convertToMb(error.size - input.max_single_file_size), 2)]) }}
                    </span>
                    <span v-if="error.validation_type === 'max_files_count_exceeded'">
                        {{(translations[error.validation_type]).replaceAll([':fileName', ':maxFilesCount'], [error.name, input.max_files_count]) }}
                    </span>
                    <span v-if="error.validation_type === 'directory'">
                        {{ translations[error.validation_type] }}
                    </span>
                </li>
            </ul>
            <h4>{{ translations.choose_file_for_upload }}</h4>
            <input type="file" multiple @change="onFileChange" :accept="getAcceptValue">
            <h5 class="text-center">{{ translations.or_drag_and_drop }}</h5>
            <!--div>
                <ul>
                    <li v-for="(file, key, index) in uploaded">
                        <img :src="getThumbnail(file.extension, key)"/>
                        <button @click="remove(key)">Remove file ({{files[key].name}})</button>
                    </li>
                </ul>
            </div-->
            <div class="row">
                <div class="col-xs-2" v-for="(file, hashKey, index) in attachments.uploaded">
                    <div class="file-thumbnail">
                        <img :src="getThumbnail(file.extension, hashKey)" class="img-responsive"/>
                        <span class="glyphicon glyphicon-zoom-in" onclick="alert('zoom-in')"></span>
                        <span class="glyphicon glyphicon-trash" @click="remove(hashKey)"></span>
                        <span>{{attachments.files[hashKey].name}}</span>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
    export default {
        props: {
            prop_component_id: {
                type: String,
                required: false
            },
            prop_translations: {
                type: Object,
                default: function () {
                    return {}
                },
                required: false
            },
            prop_csrf_name: {
                type: String,
                default: "ewis_csrf",
                required: false
            },
            prop_csrf_token: {
                type: String,
                default: (document.querySelector('input[name="ewis_csrf"]') !== null) ? document.querySelector('input[name="ewis_csrf"]').value : 'create hidden input with name "ewis_csrf" and value = csrf_hash or pass csrf props to component',
                required: false
            },
            prop_upload_link: {
                type: String,
                default: "/online/health_claim/upload",
                required: false
            },
            prop_remove_link: {
                type: String,
                default: "/online/health_claim/remove",
                required: false
            },
            prop_delete_physical: {
                type: Boolean,
                default: true,
                required: false
            },
            prop_accept: {
                type: String,
                default: "extensions",
                required: false
            },
            prop_mimes: {
                type: Array,
                default: function () {
                    return [
                        'image/jpeg',
                        'image/png',
                        'application/pdf',
                        // 'application/msword',
                        // 'application/vnd.openxmlformats-officedocument.wordprocessingml.document'
                        // 'application/vnd.ms-excel'
                        // 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
                        // 'application/x-zip-compressed',
                    ]
                },
                required: false
            },
            prop_extensions: {
                type: Array,
                default: function () {
                    return [
                        'jpeg',
                        'jpg',
                        'png',
                        'pdf',
                        // 'doc',
                        // 'docx',
                        // 'xls',
                        // 'xlsx',
                        // 'zip',
                        // 'rar',
                    ]
                },
                required: false
            },
            prop_thumbnails: {
                type: Object,
                default: function () {
                    return {
                        'pdf': '/online/images/extensions/pdf.png',
                        'doc': '/online/images/extensions/docx.png',
                        'docx': '/online/images/extensions/docx.png',
                        'xls': '/online/images/extensions/xlsx.png',
                        'xlsx': '/online/images/extensions/xlsx.png',
                        'rar': '/online/images/extensions/rar.png',
                        'zip': '/online/images/extensions/zip.png'
                    }
                },
                required: false
            },
            prop_max_single_file_size: {
                type: Number,
                default: 1000000,       // 1MB default,
                required: false
            },
            prop_max_files_count_enabled: {
                type: Boolean,
                default: false,
                required: false
            },
            prop_max_files_count: {
                type: Number,
                default: 25,            // default 25 files allowed to upload
                required: false
            },
            prop_files: {
                type: Object,
                default: function () {
                    return {}
                },
                required: false
            },
            prop_uploaded: {
                type: Object,
                default: function () {
                    return {}
                },
                required: false
            },
            prop_temp: {
                type: Object,
                default: function () {
                    return {}
                },
                required: false
            },
            prop_errors: {
                type: Array,
                default: function () {
                    return []
                },
                required: false
            },
            prop_debug: {
                type: Boolean,
                default: false,
                required: false
            },
            //  calbacks
            cb_before_upload: {
                type: String,
                default: '',
                required: false
            },
            cb_after_upload: {
                type: String,
                default: '',
                required: false
            },
            cb_before_remove: {
                type: String,
                default: '',
                required: false
            },
            cb_after_remove: {
                type: String,
                default: '',
                required: false
            }
        },
        data: function () {
            return {
                //  dev - testing
                debug: this.prop_debug,

                //  callbacks
                callbacks: {
                    beforeUpload: this.cb_before_upload,
                    afterUpload: this.cb_after_upload,
                    beforeRemove: this.cb_before_remove,
                    afterRemove: this.cb_after_remove,
                },

                //  translations
                translations: this.setTranslations(this.prop_translations),

                //  mandatory request data
                api: {
                    csrf_name: this.prop_csrf_name,                             //  CSRF token name
                    csrf_token: this.prop_csrf_token,                           //  CSRF token hash
                    upload_url: this.prop_upload_link,                          //  API upload url
                    remove_url: this.prop_remove_link,                          //  API remove url
                    delete_from_server: this.prop_delete_physical,              //  Allow (delete content from server) or disallow (save content on server) file delete
                },

                input: {
                    accept: this.prop_accept,                                   //  for input[type=file] accept attribute
                    mimes: this.prop_mimes,                                     //  allowed mime types
                    extensions: this.prop_extensions,                           //  allowed extensions if mime not found
                    thumbnails: this.prop_thumbnails,                           //  thumbnail mapping (extension:imagePath)
                    max_single_file_size: this.prop_max_single_file_size,       //  Max single file size
                    max_files_count_enabled: this.prop_max_files_count_enabled, //  Enable/disable max files count check
                    max_files_count: this.prop_max_files_count,                 //  Max files count
                },

                //  assigned with props (in future can load data from saved json objects and show last user session with files, errors...)
                attachments: {
                    files: this.prop_files,                                     //  dragged/selected files - used for image display (base64 readerResults)
                    uploaded: this.prop_uploaded,                               //  uploaded files - returned by api (uploaded 99%)
                    temp: this.prop_temp,                                       //  disallow dublicate upload
                    errors: this.prop_errors                                    //  errors array
                },
            }
        },
        methods: {
            onFileChange(e) {
                //
                var inst = this;
                //  reset errors array
                inst.attachments.errors = []
                //
                var files = e.target.files || e.dataTransfer.files;
                //
                if (!files.length) {
                    return;
                }
                //
                for (var i = 0; i < files.length; i++) {
                    // check file
                    inst.readFile(files[i]);
                }

            },
            readFile(file) {
                //
                var inst = this;
                //
                var reader = new FileReader();
                var readerResult = '';
                var mime = '';
                reader.onloadend = (e) => {
                    //  testing
                    if (inst.debug) {
                        console.log("_____");
                        console.log("%cselected/droped file ('" + file.name + "'):", 'font-weight: bold;');
                        console.log(file);
                        console.log("%cvalidation:", 'font-weight: bold;');
                    }
                    //
                    readerResult = reader.result;
                    //
                    if (readerResult === null) {
                        //  if folder draged (not file)
                        var directory = file;
                        (inst.attachments.errors).push({
                            directory: directory,
                            name: directory.name,
                            size: directory.size,
                            validation_type: 'directory'
                        });
                        //  testing
                        if (inst.debug) {
                            console.log('%cdirecotry not allowed for upload', 'color: red;')
                            console.log("_____");
                        }
                        return;
                    }
                    mime = inst.getMimeType(readerResult);
                    //  file size validation
                    if (file.size > inst.input.max_single_file_size) {
                        //  testing
                        if (inst.debug) {
                            console.log('%csingle file size is not valid (exceeds: ' + (file.size - inst.input.max_single_file_size) / 1024 / 1024 + 'MB)', 'color: red;')
                            console.log("_____");
                        }
                        //
                        (inst.attachments.errors).push({
                            file: file,
                            name: file.name,
                            type: (file.type !== '') ? file.type : mime,
                            size: file.size,
                            validation_type: 'single_size'
                        });
                    } else {
                        //  testing
                        if (inst.debug) {
                            console.log('%csingle file size is valid', 'color: green;')
                        }
                        //  file type validation
                        if ((inst.input.mimes).indexOf(mime) != -1 || (inst.input.mimes).indexOf(file.type) != -1) {
                            //  testing
                            if (inst.debug) {
                                console.log('%cfile type is valid', 'color: green;')
                            }
                            //
                            inst.pushToUpload(file, readerResult);
                        } else {
                            //  testing
                            if (inst.debug) {
                                console.log('%cfile type "%c' + mime + '%c" is not allowed for upload', 'color: red;', 'font-weight: bold; color: red;', 'color: red;');
                                console.log("_____");
                            }
                            //
                            (inst.attachments.errors).push({
                                file: file,
                                name: file.name,
                                type: (file.type !== '') ? file.type : mime,
                                size: file.size,
                                validation_type: 'mime'
                            });
                        }
                    }

                };
                //  catch IE bug (SCRIPT5022: NotSupportedError)
                try {
                    //
                    reader.readAsDataURL(file);
                } catch (e) {
                    //  EDGE directory droped
                    (inst.attachments.errors).push({
                        validation_type: 'directory'
                    });
                }

            },
            getMimeType(readerResult) {
                //  return file MIME type
                var start_position = 5;
                return readerResult.substr(start_position, (readerResult.indexOf(";") - start_position));
            },
            pushToUpload(file, readerResult) {
                //
                var inst = this;
                //
                if (!inst.input.max_files_count_enabled || (inst.input.max_files_count_enabled && (inst.input.max_files_count > Object.keys(inst.attachments.files).length))) {
                    //  push file into array
                    this.attachments.files[Math.random().toString(36).substring(7)] = {
                        lastModified: file.lastModified,
                        name: file.name,
                        size: file.size,
                        type: (file.type !== '') ? file.type : this.getMimeType(readerResult),
                        readerResult: readerResult
                    };
                    //  testing
                    if (this.debug) {
                        console.log("%cpushed to upload:", 'font-weight: bold;');
                        console.log(file);
                    }
                    //
                    this.upload();
                } else {
                    //  testing
                    if (inst.debug) {
                        console.log('%ccan\'t upload file "' + file.name + '" because uploaded files max count limit exceeded (max files: ' + inst.input.max_files_count + ')', 'color: red;')
                    }
                    //
                    (inst.attachments.errors).push({
                        file: file,
                        name: file.name,
                        type: (file.type !== '') ? file.type : 'mime-not-found',
                        size: file.size,
                        validation_type: 'max_files_count_exceeded'
                    });
                }
            },
            makeReactive() {
                //
                this.attachments.uploaded = Object.assign({}, this.attachments.uploaded, this.attachments.uploaded)
                // !!! update parent - not the best way to update
                this.$emit('parent_sync', this.$data)
            },
            remove(hashKey) {
                //
                var inst = this;

                //  testing
                if (inst.debug) {
                    console.log("_____");
                    console.log("%cremove file:", 'font-weight: bold;');
                    console.log({hashKey: inst.attachments.files[hashKey]});
                }

                //  callback before
                if (typeof window[this.callbacks.beforeRemove] === "function") {
                    window[this.callbacks.beforeRemove]()
                }

                //  allow deleting
                if (inst.api.delete_from_server) {
                    //  api request - delete file from server
                    var formData = new FormData();
                    formData.append("file", JSON.stringify(inst.attachments.uploaded[hashKey]));

                    //  pass csrf
                    if (inst.api.csrf_name !== '' && inst.api.csrf_token !== '') {
                        formData.append(inst.api.csrf_name, inst.api.csrf_token);
                    }

                    //  request
                    inst.$http.post(inst.api.remove_url, formData).then(function (response) {
                        //  testing
                        if (inst.debug) {
                            console.log("%capi response:", 'font-weight: bold;');
                            console.log(response.body);
                            console.log("_____");
                        }

                        //  callback after
                        if (typeof window[this.callbacks.afterRemove] === "function") {
                            window[this.callbacks.afterRemove]()
                        }
                    }, function (response) {
                        //
                    });
                }

                //
                delete inst.attachments.files[hashKey];  //  object
                delete inst.attachments.uploaded[hashKey];  //  object
                delete inst.attachments.temp[hashKey];  //  object

                //
                inst.makeReactive();

                //
                if(!inst.api.delete_from_server) {
                    //  callback after
                    if (typeof window[this.callbacks.afterRemove] === "function") {
                        window[this.callbacks.afterRemove]()
                    }
                }
            },
            upload() {
                //
                var inst = this;

                //  Diff
                var recentlyAddedFiles = {};
                //
                Object.keys(inst.attachments.files).forEach(function (key, index) {
                    if (!inst.attachments.temp[key]) {
                        //
                        recentlyAddedFiles[key] = inst.attachments.files[key];
                        //
                        inst.attachments.temp[key] = {};
                    } else {
                        //
                        // console.log("Fails '" + inst.attachments.uploaded[key].name + "' netika augšupielādēts, jo eksistē!");
                    }
                });

                //  payload
                var formData = new FormData();
                formData.append("files", JSON.stringify(recentlyAddedFiles));

                //  pass csrf
                if (inst.api.csrf_name !== '' && inst.api.csrf_token !== '') {
                    formData.append(inst.api.csrf_name, inst.api.csrf_token);
                }

                //  callback before
                if (typeof window[this.callbacks.beforeUpload] === "function") {
                    window[this.callbacks.beforeUpload]()
                }

                //  upload
                var fileResponse;
                inst.$http.post(inst.api.upload_url, formData).then(function (response) {
                    // response = JSON.parse(response);
                    // if (response.body) {
                    //  uploaded files response
                    Object.keys(response.body).forEach(function (key, index) {
                        //
                        fileResponse = response.body[key];
                        inst.attachments.uploaded[key] = fileResponse;
                    });
                    // } else {
                    // console.log(response.body.error); // IE11 - Unhandled promise rejection TypeError: Unable to get property 'error' of undefined or null reference
                    // }

                    //  callback after
                    if (typeof window[this.callbacks.afterUpload] === "function") {
                        window[this.callbacks.afterUpload]()
                    }

                    //  activate reactivity
                    inst.makeReactive();
                }, function (response) {
                    //
                });

                //  testing
                if (inst.debug) {
                    console.log("%cuploaded (check api response)", 'font-weight: bold;');
                    console.log("_____");
                }
            },
            onDrop(e) {
                // instance
                var inst = this;
                //  testing
                if (inst.debug) {
                    console.log('%cFile(s) dropped', 'font-weight: bold;');
                }
                //  reset errors array
                inst.attachments.errors = []
                // Prevent default behavior (Prevent file from being opened)
                e.preventDefault();
                //
                if (e.dataTransfer.items) {
                    // Use DataTransferItemList interface to access the file(s)
                    for (var i = 0; i < e.dataTransfer.items.length; i++) {
                        // If dropped items aren't files, reject them
                        if (e.dataTransfer.items[i].kind === 'file') {
                            var file = e.dataTransfer.items[i].getAsFile();
                            //
                            inst.readFile(file);
                            //  testing
                            if (inst.debug) {
                                console.log('... file[' + i + '].name = ' + file.name);
                            }
                        }
                    }
                } else {
                    //  IE10
                    if (e.dataTransfer.files.length > 0) {
                        // Use DataTransfer interface to access the file(s)
                        for (var i = 0; i < e.dataTransfer.files.length; i++) {
                            var file = e.dataTransfer.files[i];
                            //
                            inst.readFile(file);
                            //  testing
                            if (inst.debug) {
                                console.log('... file[' + i + '].name = ' + e.dataTransfer.files[i].name);
                            }
                        }
                    } else {
                        //  IE10 directory droped
                        (inst.attachments.errors).push({
                            validation_type: 'directory'
                        });
                    }
                }

                // Pass event to removeDragData for cleanup
                inst.removeDragData(e)

                //
                // document.querySelector('#' + this.prop_component_id).style.background = 'transparent';
            },
            onDragEnter(e) {
                // Prevent default behavior (Prevent file from being opened)
                e.preventDefault();
            },
            onDragLeave(e) {
                // Prevent default behavior (Prevent file from being opened)
                e.preventDefault();
                //
                // document.querySelector('#' + this.prop_component_id).style.background = 'transparent';
            },
            onDragOver(e) {
                // Prevent default behavior (Prevent file from being opened)
                e.preventDefault();
                //
                // document.querySelector('#' + this.prop_component_id).style.background = '#f9f9f9';
            },
            removeDragData(ev) {
                //  testing
                if (this.debug) {
                    console.log('%cRemoving drag data', 'font-weight: bold;');
                }
                //
                if (ev.dataTransfer.items) {
                    // Use DataTransferItemList interface to remove the drag data
                    ev.dataTransfer.items.clear();
                } else {
                    // Use DataTransfer interface to remove the drag data
                    ev.dataTransfer.clearData();
                }
            },
            getThumbnail(extension, fileKey) {
                //  default
                var src = '/online/images/extensions/unknown.png'
                //
                if (extension in this.input.thumbnails) {
                    //  if extension exists in object
                    src = this.input.thumbnails[extension]
                } else {
                    //
                    switch ((extension).toLowerCase()) {
                        case 'png':
                        case 'jpg':
                        case 'jpeg':
                        case 'gif':
                        case 'svg':
                            src = this.attachments.files[fileKey].readerResult;
                            break;
                    }
                }
                //
                return src;
            },
            precisionRound(number, precision) {
                var factor = Math.pow(10, precision);
                return Math.round(number * factor) / factor;
            },
            convertToKb(size) {
                return size / 1024
            },
            convertToMb(size) {
                return size / 1024 / 1024
            },
            setTranslations(overwrite) {
                var def = {
                    choose_file_for_upload: "Choose documents for upload",
                    or_drag_and_drop: "or drag & drop here",
                    mime: 'Document ":fileName" can\'t be uploaded, file type ":fileType" is not allowed',
                    single_size: 'Document ":fileName" can\'t be uploaded, file size exceeds maximum file size ":fileMaxSize Mb" by ":fileSize Mb"',
                    max_files_count_exceeded: 'Document ":fileName" can\'t be uploaded, maximum number of files reached ":maxFilesCount"',
                    directory: 'Directory can not be processed, please select the files',
                }
                //
                Object.assign(def, overwrite);
                //
                return Object.assign({}, overwrite, def);
            }
        },
        // watch: {
        //     'attachments.errors'() {
        //         setTimeout(() => {
        //             this.attachments.errors = [];
        //         }, 5000)
        //     }
        // },
        computed: {
            getAcceptValue() {
                var accept = this.input.accept;
                if (accept === 'mimes') {
                    return this.input[accept].join()
                }
                if (accept === 'extensions') {
                    return '.' + this.input[accept].join(',.')
                }
            }
        },
        beforeCreate() {
            String.prototype.replaceAll = function (search_array, replacement_array) {
                //
                var target = this;
                //
                search_array.forEach(function (substr, index) {
                    if (typeof replacement_array[index] != "undefined") {
                        target = target.replace(new RegExp(substr, 'g'), replacement_array[index])
                    }
                });
                //
                return target;
            };
        },
        mounted: function () {
            // console.log(this.$parent.$options.methods.test());
            console.info("upload component ('#" + this.prop_component_id + "') loaded. Use attribute @parent_sync=\"[parentMethodName]\" to sync parent object with child. For cb pass function name.");
        }
    }
</script>

<style scoped>
    .v-grag-n-drop-container {
        border: 1px dashed black;
        padding: 40px 20px;
    }

    img {
        border-bottom: 1px solid #333333;
    }

    .file-thumbnail {
        position: relative;
        border: 1px solid #333333;
    }

    .glyphicon {
        position: absolute;
        background: #CCCCCC;
        color: #FFFFFF !important;
        font-size: 18px;
        padding: 5px;
        cursor: pointer;
    }

    .glyphicon-trash {
        position: absolute;
        top: 0px;
        right: 0px;
        border-left: 1px solid #333333;
        border-bottom: 1px solid #333333;
    }

    .glyphicon-zoom-in {
        top: 0px;
        left: 0px;
        border-right: 1px solid #333333;
        border-bottom: 1px solid #333333;
    }

</style>