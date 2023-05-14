<script lang="ts">
 import UploadText from "$lib/components/gradio/app/UploadText.svelte";
 import type { ComfyImageLocation } from "$lib/nodes/ComfyWidgetNodes";
 import notify from "$lib/notify";
 import { convertComfyOutputEntryToGradio, convertComfyOutputToComfyURL, type ComfyUploadImageAPIResponse } from "$lib/utils";
 import { Block, BlockLabel } from "@gradio/atoms";
 import { File as FileIcon } from "@gradio/icons";
 import type { FileData as GradioFileData } from "@gradio/upload";
 import { ModifyUpload, Upload } from "@gradio/upload";
 import { createEventDispatcher, tick } from "svelte";

 export let value: ComfyImageLocation[] | null = null;
 export let imgWidth: number = 0;
 export let imgHeight: number = 0;
 export let imgElem: HTMLImageElement | null = null
 export let fileCount: "single" | "multiple" = "single"
 export let elem_classes: string[] = []
 export let style: string = ""
 export let label: string = ""
 // let propsChanged: Writable<number> | null = null;
 let dragging = false;
 let pending_upload = false;
 let old_value: GradioFileData[] | null = null;

 let _value: GradioFileData[] | null = null;
 const root = "comf"
 const root_url = "https//ComfyUI!"
 let uploaded: boolean = false;

	const dispatch = createEventDispatcher<{
		change: ComfyImageLocation[];
        uploading: undefined;
		uploaded: ComfyImageLocation[];
		upload_error: any;
		clear: undefined;
		image_clicked: undefined;
	}>();

 if (value) {
     _value = null
     if (imgElem)
         imgElem.src = convertComfyOutputToComfyURL(value[0])
 }

 $: if (!(_value && _value.length > 0 && imgElem)) {
     imgWidth = 1
     imgHeight = 1
 }

 function onChange() {
     dispatch("change", value)
 }

 function onUpload() {
     dispatch("uploaded")
 }

 function onClear() {
     dispatch("clear")
 }

 function onImgClicked() {
     dispatch("image_clicked")
 }

 interface GradioUploadResponse {
     error?: string;
     files?: Array<ComfyImageLocation>;
 }

 async function upload_files(root: string, files: Array<File>): Promise<GradioUploadResponse> {
     console.debug("UPLOADILFES", root, files);

     dispatch("uploading")

     const url = `http://${location.hostname}:8188` // TODO make configurable

     const requests = files.map(async (file) => {
         const formData = new FormData();
         formData.append("image", file, file.name);
         return fetch(new Request(url + "/upload/image", {
             body: formData,
             method: 'POST'
         }))
         .then(r => r.json())
         .catch(error => error);
     });

     return Promise.all(requests)
                   .then( (results) => {
                       const errors = []
                       const files = []

                       for (const r of results) {
                           if (r instanceof Error) {
                               errors.push(r.cause)
                           }
                           else {
                               // bare filename of image
                               const resp = r as ComfyUploadImageAPIResponse;
                               files.push({
                                   filename: resp.name,
                                   subfolder: "",
                                   type: "input"
                               })
                           }
                       }

                       let error = null;
                       if (errors && errors.length > 0)
                           error = "Upload error(s):\n" + errors.join("\n");

                       return { error, files }
                   })
 }

 $: {
     if (JSON.stringify(_value) !== JSON.stringify(old_value) || uploaded) {
         uploaded = false;
         pending_upload = true;

         old_value = _value;

         if (_value == null)
             _value = []
         else if (!Array.isArray(_value))
             _value = [_value]

         const allBlobs = _value.map((file_data: GradioFileData) => file_data.blob)

         if (allBlobs == null || allBlobs.length === 0) {
             _value = null;
             value = null;
             onChange();
             pending_upload = false;
         }
         else if (!allBlobs.every(b => b != null)) {
             _value = null;
             value = null;
             onChange();
             pending_upload = false;
         }
         else {
             let files = (Array.isArray(_value) ? _value : [_value]).map(
                 (file_data) => file_data.blob!
             );
             let upload_value = _value;
             pending_upload = true;
             upload_files(root, files).then((response) => {
                 if (JSON.stringify(upload_value) !== JSON.stringify(_value)) {
                     // value has changed since upload started
                     console.error("[ImageUpload] value has changed since upload started", upload_value, _value)
                     return;
                 }

                 pending_upload = false;

                 if (response.error) {
                     notify(response.error, { type: "error" })
                 }

                 value = response.files;
                 dispatch("change", value)
                 dispatch("uploaded", value)
             }).
             catch(err => {
                 dispatch("upload_error", err)
             });
         }
     }
 }

 async function handle_upload({ detail }: CustomEvent<GradioFileData | Array<GradioFileData>>) {
     // Received Gradio-format file data from the Upload component.
     // In the reactive block above it will be uploaded to ComfyUI.
     _value = Array.isArray(detail) ? detail : [detail];
     uploaded = true;
 }

 function handle_clear(_e: CustomEvent<null>) {
     _value = null;
     value = [];
     dispatch("change", value)
     dispatch("clear")
 }

 function convertGradioUpload(e: CustomEvent<GradioFileData[]>) {
     _value = e.detail
 }
</script>

<div class="image-upload" {style}>
    {#if value}
        <Block
            visible={true}
            variant={(value === null || value.length === 0) ? "dashed" : "solid"}
            border_mode={dragging ? "focus" : "base"}
            padding={true}
            elem_id="comfy-image-upload-block"
            {elem_classes}
        >
            <BlockLabel
                label={label}
                show_label={label != ""}
                Icon={FileIcon}
                float={label != ""}
            />
            {#if value && value.length > 0 && !pending_upload}
                {@const firstImage = value[0]}
                <ModifyUpload on:clear={handle_clear} absolute />
                <img src={convertComfyOutputToComfyURL(firstImage)}
                     alt={firstImage.filename}
                     on:click={onImgClicked}
                     bind:this={imgElem}
                     bind:naturalWidth={imgWidth}
                     bind:naturalHeight={imgHeight}
                />
            {:else}
                <Upload
                    file_count={fileCount}
                    filetype="image/*"
                    on:change={convertGradioUpload}
                    on:load={handle_upload}
                    bind:dragging
                    on:clear
                    on:select
                    parse_to_data_url={false}
                >
                    <UploadText type="file" />
                </Upload>
            {/if}
        </Block>
    {/if}
</div>

<style lang="scss">
 .image-upload {
     height: var(--size-96);

     :global(.block) {
         height: inherit;
         padding: 0;
         border-radius: 0;
     }

     img {
         width: 100%;
         height: 100%;
         max-width: 100%;
         max-height: 100%;
         object-fit: contain;
     }
 }
</style>