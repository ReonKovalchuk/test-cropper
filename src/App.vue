<template>
    <div class="cropper">
        <div class="cropper__btns">
            <button
                class="cropper__btn"
                :disabled="!selectionActive"
                @click="removeSelectedArea"
            >
                Убрать выбранную область
            </button>

            <button
                :disabled="!selectionActive"
                class="cropper__btn"
                @click="getCroppedCoords"
            >
                Сохранить
            </button>
        </div>

        <div v-if="!selectedFile">
            <input
                ref="input"
                type="file"
                accept="image/*"
                class="cropper__input"
                @change="handleFileChange"
            />
            <button
                class="cropper__btn"
                @click="triggerFileInput"
            >
                Выберите файл
            </button>
        </div>

        <div
            v-else
            class="cropper__inner"
        >
            <img
                ref="image"
                :src="selectedFile"
                class="cropper__img"
                draggable="false"
                @pointerdown="handleMouseDown"
                @pointermove="handleMouseMove"
                @pointerup="handleMouseUp"
            />
            <div
                class="cropper__selected"
                :class="{ _active: selectionActive }"
                ref="selectedAreaEl"
            >
                <div
                    class="cropper__control _tl"
                    :class="{ _disabled: selecting || controlActive }"
                    @pointerdown="
                        (event) => handleMouseDownControl(event, 'tl')
                    "
                ></div>
                <div
                    class="cropper__control _tr"
                    :class="{ _disabled: selecting || controlActive }"
                    @pointerdown="
                        (event) => handleMouseDownControl(event, 'tr')
                    "
                ></div>
                <div
                    class="cropper__control _bl"
                    :class="{ _disabled: selecting || controlActive }"
                    @pointerdown="
                        (event) => handleMouseDownControl(event, 'bl')
                    "
                ></div>
                <div
                    class="cropper__control _br"
                    :class="{ _disabled: selecting || controlActive }"
                    @pointerdown="
                        (event) => handleMouseDownControl(event, 'br')
                    "
                ></div>
            </div>
        </div>
        <div v-if="resCoords">
            top left point: {{ resCoords[0] }}
            <br />
            bottom right point:
            {{ resCoords[1] }}
        </div>
    </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'

interface Coords {
    x: number
    y: number
}

type ControlIds = 'tl' | 'tr' | 'br' | 'bl'
type CornerCords = {
    [key in ControlIds]: Coords
}

const SELECTED_AREA_THRESHOLD = 10

const selectedFile = ref<string>('')
const selectedAreaEl = ref<HTMLElement | null>(null)
const input = ref<HTMLElement | null>(null)
const image = ref<HTMLImageElement | null>(null)
const diagonalCoords = ref<[Coords, Coords]>()
const cornerCoords = ref<CornerCords>()
const selecting = ref<boolean>(false)
const selectionActive = ref<boolean>(false)
const controlActive = ref<string>()
const resCoords = ref<[Coords, Coords]>()

function triggerFileInput() {
    input.value?.click()
}

function handleFileChange(event: Event) {
    const target = event.target as HTMLInputElement
    const file = target.files?.[0]

    if (file) {
        const reader = new FileReader()
        reader.onload = (e) => {
            selectedFile.value = e.target?.result as string
        }
        reader.readAsDataURL(file)
    }
}

function handleMouseDown(event: MouseEvent) {
    if (!image.value) return

    if (controlActive.value) {
        controlActive.value = undefined
    } else if (selecting.value) {
        selecting.value = false
    } else {
        selecting.value = true

        const { x, y } = getRelativePosition(event, image.value)
        diagonalCoords.value = [
            { x, y },
            { x, y },
        ]
        selectionActive.value = true
    }
}

function handleMouseMove(event: MouseEvent) {
    if (!image.value || !diagonalCoords.value) return

    if (selecting.value) {
        const { x, y } = getRelativePosition(event, image.value)
        diagonalCoords.value[1] = { x, y }
        updateArea(diagonalCoords.value)
    } else if (controlActive.value) {
        handleMouseMoveControl(event)
    }
}

function handleMouseUp() {
    if (selecting.value) {
        if (
            selectedAreaEl.value?.offsetWidth
            || 0 > SELECTED_AREA_THRESHOLD
            || selectedAreaEl.value?.offsetHeight
            || 0 > SELECTED_AREA_THRESHOLD
        ) {
            selecting.value = false
        }
    } else if (controlActive.value) {
        controlActive.value = undefined
    }
}

function getRelativePosition(event: MouseEvent, image: HTMLElement): Coords {
    const rect = image.getBoundingClientRect()
    const x = event.clientX - rect.left + image.offsetLeft //x position within the element.
    const y = event.clientY - rect.top + image.offsetTop

    return { x, y }
}

function updateArea(coords: [Coords, Coords]) {
    const minX = Math.min(coords[0].x, coords[1].x)
    const minY = Math.min(coords[0].y, coords[1].y)
    const maxX = Math.max(coords[0].x, coords[1].x)
    const maxY = Math.max(coords[0].y, coords[1].y)

    updateSelectedArea({ x: minX, y: minY }, { x: maxX, y: maxY })
}

function removeSelectedArea() {
    selecting.value = false
    selectionActive.value = false
    cornerCoords.value = undefined
    diagonalCoords.value = undefined
    resCoords.value = undefined
}

function handleMouseDownControl(event: MouseEvent, controlId: ControlIds) {
    resCoords.value = undefined
    controlActive.value = controlId
    handleMouseMoveControl(event)
}

function handleMouseMoveControl(event: MouseEvent) {
    if (!controlActive.value || !image.value || !cornerCoords.value) return

    const { x, y } = getRelativePosition(event, image.value)

    switch (controlActive.value) {
        case 'tl':
            cornerCoords.value.tl = { x, y }
            cornerCoords.value.bl.x = x
            cornerCoords.value.tr.y = y
            break
        case 'tr':
            cornerCoords.value.tr = { x, y }
            cornerCoords.value.br.x = x
            cornerCoords.value.tl.y = y
            break
        case 'br':
            cornerCoords.value.br = { x, y }
            cornerCoords.value.tr.x = x
            cornerCoords.value.bl.y = y
            break
        case 'bl':
            cornerCoords.value.bl = { x, y }
            cornerCoords.value.tl.x = x
            cornerCoords.value.br.y = y
            break
        default:
            break
    }

    updateSelectedArea(cornerCoords.value.tl, cornerCoords.value.br)
}

function updateSelectedArea(tl: Coords, br: Coords) {
    if (!selectedAreaEl.value) return

    const width = br.x - tl.x
    const height = br.y - tl.y

    cornerCoords.value = {
        tl,
        tr: { x: br.x, y: tl.y },
        br,
        bl: { x: tl.x, y: br.y },
    }

    selectedAreaEl.value.style.top = `${tl.y}px`
    selectedAreaEl.value.style.left = `${tl.x}px`
    selectedAreaEl.value.style.width = `${width}px`
    selectedAreaEl.value.style.height = `${height}px`
}

function getCroppedCoords() {
    if (!image.value) return
    const zoom = image.value.clientHeight / image.value.naturalHeight
    const offsetTop = image.value.offsetTop
    const offsetLeft = image.value.offsetLeft
    resCoords.value = [
        {
            x: Math.round(
                ((cornerCoords.value?.tl.x || 0) - offsetLeft) / zoom,
            ),
            y: Math.round(((cornerCoords.value?.tl.y || 0) - offsetTop) / zoom),
        },
        {
            x: Math.round(
                ((cornerCoords.value?.br.x || 0) - offsetLeft) / zoom,
            ),
            y: Math.round(((cornerCoords.value?.br.y || 0) - offsetTop) / zoom),
        },
    ]
}
</script>

<style scoped lang="scss">
.cropper {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    gap: 30px;
    margin: 0 auto;

    &__input {
        display: none;
    }

    &__btns {
        display: flex;
        flex-direction: row;
        justify-content: space-between;
        align-self: stretch;
        gap: 12px;
        flex-wrap: wrap;
    }

    &__btn {
        padding: 12px;
        font-size: 20px;
    }

    &__inner {
        background-color: white;
        border: 1px solid black;
        width: 800px;
        height: 600px;
        max-width: 100%;
        max-height: 100%;
        position: relative;
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: center;
        overflow: hidden;
    }

    &__img {
        object-fit: contain;
        max-width: 100%;
        max-height: 100%;
        user-select: none;
        user-drag: none;
        -webkit-user-drag: none;
        user-select: none;
        -moz-user-select: none;
        -webkit-user-select: none;
        -ms-user-select: none;
    }

    &__selected {
        border: 1px solid black;
        position: absolute;
        display: none;
        z-index: 1;
        pointer-events: none;

        &._active {
            display: block;
        }
        &:after {
            content: '';
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            box-shadow: 0px 0px 0px 9999px rgba(228, 228, 228, 0.5);
        }
    }

    &__control {
        --control-size: 20px;
        --control-offset: calc(-1 * var(--control-size) / 2);

        background-color: aquamarine;
        border-radius: 50%;
        aspect-ratio: 1;
        width: var(--control-size);
        position: absolute;
        z-index: 2;
        pointer-events: all;

        &._disabled {
            pointer-events: none;
        }

        &._tl {
            top: var(--control-offset);
            left: var(--control-offset);
        }
        &._tr {
            top: var(--control-offset);
            right: var(--control-offset);
        }
        &._br {
            bottom: var(--control-offset);
            right: var(--control-offset);
        }
        &._bl {
            bottom: var(--control-offset);
            left: var(--control-offset);
        }
    }
}
</style>
