<template>
    <div class="annotation-tool">
        <div class="toolbar">
            <button @click="setMode('select')" :class="{ active: mode === 'select' }">选择</button>
            <button @click="setMode('rect')" :class="{ active: mode === 'rect' }">矩形</button>
            <button @click="setMode('polygon')" :class="{ active: mode === 'polygon' }">多边形</button>
            <button @click="setMode('polyline')" :class="{ active: mode === 'polyline' }">折线</button>
            <button @click="setMode('point')" :class="{ active: mode === 'point' }">点</button>
            <button @click="deleteSelected" :disabled="!selectedId">删除</button>
            <button @click="undoLastPoint" :disabled="!canUndoPoint">撤销点</button>
            <button @click="exportAnnotations">导出</button>
            <!-- <button @click="loadImage">加载图片</button> -->
            <button @click="resetView">重置视图</button>
        </div>

        <div class="main-content">
            <div class="left-panel">
                <!-- 图片列表 -->
                <div class="image-list">
                    <h3>图片列表</h3>
                    <div class="image-items">
                        <div v-for="(img, index) in imageList" :key="index"
                            :class="{ 'image-item': true, 'active': currentImageIndex === index }"
                            @click="selectImage(index)">
                            <div class="image-thumbnail">
                                <img :src="img.url" alt="图片" />
                            </div>
                            <div class="image-info">
                                <div class="image-name">{{ img.filename }}</div>
                                <div class="annotation-count">{{ getImageAnnotationCount(index) }} 个标注</div>
                            </div>
                        </div>
                        <div v-if="imageList.length === 0" class="no-images">
                            暂无图片
                        </div>
                    </div>
                    <div class="image-actions">
                        <button @click="addImage" class="add-image-btn">添加图片</button>
                        <button @click="removeCurrentImage" :disabled="currentImageIndex === -1"
                            class="remove-image-btn">删除当前图片</button>
                    </div>
                </div>

                <!-- 标签列表 -->
                <div class="label-list">
                    <h3>标签列表</h3>
                    <div class="label-items">
                        <div v-for="label in uniqueLabels" :key="label"
                            :class="{ 'label-item': true, 'active': isLabelActive(label) }"
                            @click="selectAnnotationByLabel(label)">
                            {{ label }}
                            <span class="label-count">({{ getLabelCount(label) }})</span>
                        </div>
                        <div v-if="uniqueLabels.length === 0" class="no-labels">
                            暂无标签
                        </div>
                    </div>
                </div>
            </div>

            <div class="canvas-container" ref="containerRef">
                <canvas ref="canvasRef" @mousedown="handleMouseDown" @mousemove="handleMouseMove"
                    @mouseup="handleMouseUp" @dblclick="handleDoubleClick" @wheel="handleWheel"
                    @contextmenu="handleRightClick"></canvas>

                <!-- 标签编辑输入框 -->
                <div v-if="selectedId && editingLabel" class="label-editor" :style="labelEditorStyle">
                    <input ref="labelInputRef" v-model="labelText" @blur="saveLabel" @keydown.enter="saveLabel"
                        placeholder="输入标签" />
                </div>

                <!-- 标签属性编辑器 -->
                <div v-if="selectedId && editingProperties" class="properties-editor" :style="propertiesEditorStyle">
                    <div class="properties-header">
                        <h4>标签属性</h4>
                        <button @click="closePropertiesEditor" class="close-btn">×</button>
                    </div>
                    <div class="properties-content">
                        <div class="property-item" v-for="(value, key) in currentProperties" :key="key">
                            <input v-model="propertyKeys[key]" placeholder="属性名"
                                @change="updatePropertyKey(key, propertyKeys[key])" />
                            <input v-model="currentProperties[key]" placeholder="属性值"
                                @change="updatePropertyValue(key, currentProperties[key])" />
                            <button @click="removeProperty(key)" class="remove-property-btn">×</button>
                        </div>
                        <button @click="addProperty" class="add-property-btn">+ 添加属性</button>
                    </div>
                    <div class="properties-footer">
                        <button @click="saveProperties" class="save-btn">保存</button>
                        <button @click="closePropertiesEditor" class="cancel-btn">取消</button>
                    </div>
                </div>

                <!-- 点操作提示 -->
                <div v-if="pointOperationHint" class="hint">
                    {{ pointOperationHint }}
                </div>
            </div>
        </div>

        <!-- 标注属性面板 -->
        <div v-if="selectedAnnotation" class="annotation-properties-panel">
            <h3>标注属性</h3>
            <div class="annotation-info">
                <div class="info-item">
                    <span class="label">类型:</span>
                    <span class="value">{{ getAnnotationTypeName(selectedAnnotation.type) }}</span>
                </div>
                <div class="info-item">
                    <span class="label">标签:</span>
                    <span class="value">{{ selectedAnnotation.label || '无' }}</span>
                    <button @click="editSelectedAnnotationLabel" class="edit-btn">编辑</button>
                </div>
                <div class="info-item">
                    <span class="label">属性:</span>
                    <div class="properties-list">
                        <div v-if="Object.keys(selectedAnnotation.properties || {}).length === 0" class="no-properties">
                            无属性
                        </div>
                        <div v-else>
                            <div v-for="(value, key) in selectedAnnotation.properties" :key="key"
                                class="property-display">
                                <span class="property-key">{{ key }}:</span>
                                <span class="property-value">{{ value }}</span>
                            </div>
                        </div>
                    </div>
                    <button @click="editSelectedAnnotationProperties" class="edit-btn">编辑</button>
                </div>
            </div>
        </div>

        <div class="status">
            <span>当前模式: {{ modeText }}</span>
            <span v-if="selectedId">已选中: {{ selectedId }}</span>
            <span v-if="(mode === 'polygon' || mode === 'polyline') && currentPoints.length > 0">点数: {{
                currentPoints.length }}</span>
            <span>缩放: {{ Math.round(scale * 100) }}%</span>
            <span v-if="currentImageIndex >= 0">当前图片: {{ currentImageIndex + 1 }}/{{ imageList.length }}</span>
        </div>
    </div>
</template>

<script setup lang="ts">
import { ref, onMounted, computed, nextTick, watch } from 'vue'

// 标注类型定义
type AnnotationType = 'rect' | 'polygon' | 'polyline' | 'point'

interface Point {
    x: number
    y: number
}

interface BaseAnnotation {
    id: string
    type: AnnotationType
    color: string
    label?: string
    properties?: Record<string, string>
}

interface RectAnnotation extends BaseAnnotation {
    type: 'rect'
    x: number
    y: number
    width: number
    height: number
}

interface PolygonAnnotation extends BaseAnnotation {
    type: 'polygon'
    points: Point[]
}

interface PolylineAnnotation extends BaseAnnotation {
    type: 'polyline'
    points: Point[]
}

interface PointAnnotation extends BaseAnnotation {
    type: 'point'
    x: number
    y: number
}

type Annotation = RectAnnotation | PolygonAnnotation | PolylineAnnotation | PointAnnotation

interface ImageData {
    url: string
    filename: string
    annotations: Annotation[]
}

// 组件状态
const canvasRef: any = ref<HTMLCanvasElement | null>(null)
const containerRef = ref<HTMLDivElement | null>(null)
const labelInputRef = ref<HTMLInputElement | null>(null)
const mode = ref<'select' | 'rect' | 'polygon' | 'polyline' | 'point'>('select')
const annotations = ref<Annotation[]>([])
const selectedId = ref<string | null>(null)
const isDrawing = ref(false)
const startPoint = ref<Point>({ x: 0, y: 0 })
const currentPoints = ref<Point[]>([])
const tempAnnotation = ref<Annotation | null>(null)
const backgroundImage = ref<HTMLImageElement | null>(null)
const isDragging = ref(false)
const dragType = ref<'move' | 'resize' | 'vertex' | 'image' | 'addPoint' | null>(null)
const dragIndex = ref<number>(-1)
const dragOffset = ref<Point>({ x: 0, y: 0 })

// 图片视图状态
const scale = ref(1)
const offset = ref<Point>({ x: 0, y: 0 })
const isDraggingImage = ref(false)
const lastMousePos = ref<Point>({ x: 0, y: 0 })

// 标签编辑状态
const editingLabel = ref(false)
const labelText = ref('')
const labelEditorStyle = ref({ top: '0px', left: '0px' })

// 标签属性编辑状态
const editingProperties = ref(false)
const currentProperties = ref<Record<string, string>>({})
const propertyKeys: any = ref<Record<string, string>>({})
const propertiesEditorStyle = ref({ top: '0px', left: '0px' })

// 多边形/折线拖拽状态
const polygonDragStartPoints = ref<Point[]>([])

// 点操作状态
const pointOperationHint = ref<string>('')

// 图片列表状态
const imageList = ref<ImageData[]>([])
const currentImageIndex = ref(-1)

// 计算属性
const modeText = computed(() => {
    const modeMap = {
        select: '选择模式',
        rect: '绘制矩形',
        polygon: '绘制多边形',
        polyline: '绘制折线',
        point: '绘制点'
    }
    return modeMap[mode.value]
})

// 是否可以撤销点
const canUndoPoint = computed(() => {
    return (mode.value === 'polygon' || mode.value === 'polyline') && currentPoints.value.length > 0
})

// 获取所有唯一标签
const uniqueLabels = computed(() => {
    const labels = new Set<string>()
    imageList.value.forEach(imageData => {
        imageData.annotations.forEach(annotation => {
            if (annotation.label) {
                labels.add(annotation.label)
            }
        })
    })
    return Array.from(labels).sort()
})

// 获取选中的标注
const selectedAnnotation = computed(() => {
    if (!selectedId.value) return null
    return annotations.value.find(a => a.id === selectedId.value) || null
})

// 获取图片边界
const getImageBounds = () => {
    if (!backgroundImage.value) return null

    return {
        left: offset.value.x,
        top: offset.value.y,
        right: offset.value.x + backgroundImage.value.width * scale.value,
        bottom: offset.value.y + backgroundImage.value.height * scale.value,
        width: backgroundImage.value.width,
        height: backgroundImage.value.height
    }
}

// 限制点在图片内
const clampPointToImage = (point: Point): Point => {
    const bounds = getImageBounds()
    if (!bounds) return point

    return {
        x: Math.max(bounds.left, Math.min(point.x, bounds.right)),
        y: Math.max(bounds.top, Math.min(point.y, bounds.bottom))
    }
}

// 限制矩形在图片内 - 严格版本
const clampRectToImageStrict = (rect: RectAnnotation): RectAnnotation => {
    const bounds = getImageBounds()
    if (!bounds) return rect

    // 转换为画布坐标
    const canvasRect = {
        x: rect.x * scale.value + offset.value.x,
        y: rect.y * scale.value + offset.value.y,
        width: rect.width * scale.value,
        height: rect.height * scale.value
    }

    // 严格限制在图片边界内
    let clampedX = canvasRect.x
    let clampedY = canvasRect.y
    let clampedWidth = canvasRect.width
    let clampedHeight = canvasRect.height

    // 限制X坐标
    if (clampedX < bounds.left) {
        clampedX = bounds.left
    } else if (clampedX + clampedWidth > bounds.right) {
        clampedX = bounds.right - clampedWidth
    }

    // 限制Y坐标
    if (clampedY < bounds.top) {
        clampedY = bounds.top
    } else if (clampedY + clampedHeight > bounds.bottom) {
        clampedY = bounds.bottom - clampedHeight
    }

    // 如果宽度超出边界，调整宽度
    if (clampedX + clampedWidth > bounds.right) {
        clampedWidth = bounds.right - clampedX
    }

    // 如果高度超出边界，调整高度
    if (clampedY + clampedHeight > bounds.bottom) {
        clampedHeight = bounds.bottom - clampedY
    }

    // 转换回图片坐标
    return {
        ...rect,
        x: (clampedX - offset.value.x) / scale.value,
        y: (clampedY - offset.value.y) / scale.value,
        width: clampedWidth / scale.value,
        height: clampedHeight / scale.value
    }
}

// 初始化画布
onMounted(() => {
    const canvas = canvasRef.value
    const container = containerRef.value
    if (!canvas || !container) return

    // 设置画布尺寸为容器尺寸
    resizeCanvas()

    // 监听窗口大小变化
    window.addEventListener('resize', resizeCanvas)

    // 监听键盘事件
    window.addEventListener('keydown', (e) => {
        if (e.key === 'ArrowLeft') {
            if (currentImageIndex.value > 0) {
                selectImage(currentImageIndex.value - 1)
            }
        }
        if (e.key === 'ArrowRight') {
            if (currentImageIndex.value < imageList.value.length - 1) {
                selectImage(currentImageIndex.value + 1)
            }
        }
    })

    // 初始绘制
    redrawCanvas()
})

// 调整画布尺寸
const resizeCanvas = () => {
    const canvas = canvasRef.value
    const container = containerRef.value
    if (!canvas || !container) return

    canvas.width = container.clientWidth
    canvas.height = container.clientHeight

    // 如果有图片，重新计算居中位置
    if (backgroundImage.value) {
        centerImage()
    }

    // 重绘画布
    redrawCanvas()
}

// 加载图片
const loadImage = () => {
    const canvas = canvasRef.value
    if (!canvas) return

    const img = new Image()
    img.crossOrigin = 'Anonymous' // 处理跨域问题
    img.src = 'http://localhost:9000/img/1.jpg'

    img.onload = () => {
        // 添加新图片到列表
        imageList.value.push({
            url: img.src,
            filename: 'demo.jpg',
            annotations: []
        })

        // 选中新添加的图片
        selectImage(imageList.value.length - 1)
    }

    img.onerror = () => {
        console.error('图片加载失败')
        const canvas = canvasRef.value
        if (!canvas) return

        const ctx = canvas.getContext('2d')
        if (ctx) {
            ctx.fillStyle = '#f0f0f0'
            ctx.fillRect(0, 0, canvas.width, canvas.height)
            ctx.fillStyle = '#ff0000'
            ctx.font = '20px Arial'
            ctx.textAlign = 'center'
            ctx.fillText('图片加载失败', canvas.width / 2, canvas.height / 2)
        }
    }
}

// 添加图片
const addImage = () => {
    const input = document.createElement('input')
    input.type = 'file'
    input.accept = 'image/*'
    input.multiple = true

    input.onchange = (e: any) => {
        const files = e.target.files
        if (!files || files.length === 0) return

        Array.from(files).forEach((file: any) => {
            const reader = new FileReader()
            reader.onload = (e) => {
                const url = e.target?.result as string
                imageList.value.push({
                    url,
                    filename: file.name,
                    annotations: []
                })
            }
            reader.readAsDataURL(file)
        })

        // 选中新添加的第一张图片
        setTimeout(() => {
            if (imageList.value.length > 0) {
                selectImage(imageList.value.length - files.length)
            }
        }, 100)
    }

    input.click()
}

// 删除当前图片
const removeCurrentImage = () => {
    if (currentImageIndex.value === -1) return

    if (confirm('确定要删除当前图片吗？删除后所有标注将丢失。')) {
        imageList.value.splice(currentImageIndex.value, 1)

        if (imageList.value.length === 0) {
            currentImageIndex.value = -1
            backgroundImage.value = null
            annotations.value = []
            selectedId.value = null
            redrawCanvas()
        } else {
            // 选择删除位置后的图片
            selectImage(Math.min(currentImageIndex.value, imageList.value.length - 1))
        }
    }
}

// 选择图片
const selectImage = (index: number) => {
    if (index < 0 || index >= imageList.value.length) return

    // 保存当前图片的标注（如果有当前图片）
    if (currentImageIndex.value >= 0 && currentImageIndex.value < imageList.value.length) {
        imageList.value[currentImageIndex.value].annotations = [...annotations.value]
    }

    currentImageIndex.value = index
    const imageData = imageList.value[index]

    // 加载新图片
    const img = new Image()
    img.crossOrigin = 'Anonymous'
    img.src = imageData.url

    img.onload = () => {
        backgroundImage.value = img

        // 加载新图片的标注
        annotations.value = [...imageData.annotations]
        selectedId.value = null

        // 居中显示图片
        centerImage()

        // 绘制图片
        redrawCanvas()
    }

    img.onerror = () => {
        console.error('图片加载失败')
        const canvas = canvasRef.value
        if (!canvas) return

        const ctx = canvas.getContext('2d')
        if (ctx) {
            ctx.fillStyle = '#f0f0f0'
            ctx.fillRect(0, 0, canvas.width, canvas.height)
            ctx.fillStyle = '#ff0000'
            ctx.font = '20px Arial'
            ctx.textAlign = 'center'
            ctx.fillText('图片加载失败', canvas.width / 2, canvas.height / 2)
        }
    }
}

// 获取图片的标注数量
const getImageAnnotationCount = (index: number): number => {
    if (index < 0 || index >= imageList.value.length) return 0
    return imageList.value[index].annotations.length
}

// 居中图片
const centerImage = () => {
    const canvas = canvasRef.value
    const img = backgroundImage.value
    if (!canvas || !img) return

    // 计算缩放比例，使图片适应容器
    const scaleX = canvas.width / img.width
    const scaleY = canvas.height / img.height
    scale.value = Math.min(scaleX, scaleY, 1) // 不超过原始大小

    // 计算偏移量使图片居中
    const scaledWidth = img.width * scale.value
    const scaledHeight = img.height * scale.value
    offset.value = {
        x: (canvas.width - scaledWidth) / 2,
        y: (canvas.height - scaledHeight) / 2
    }
}

// 重置视图
const resetView = () => {
    if (backgroundImage.value) {
        centerImage()
        redrawCanvas()
    }
}

// 设置模式
const setMode = (newMode: typeof mode.value) => {
    mode.value = newMode
    selectedId.value = null
    currentPoints.value = []
    tempAnnotation.value = null
    isDragging.value = false
    dragType.value = null
    editingLabel.value = false
    editingProperties.value = false
    pointOperationHint.value = ''
    redrawCanvas()
}

// 撤销最后一个点
const undoLastPoint = () => {
    if ((mode.value === 'polygon' || mode.value === 'polyline') && currentPoints.value.length > 0) {
        currentPoints.value.pop()
        pointOperationHint.value = '已撤销最后一个点'
        setTimeout(() => {
            pointOperationHint.value = ''
        }, 2000)
        redrawCanvas()
    }
}

// 坐标转换：画布坐标 -> 图片原始坐标
const canvasToImage = (canvasX: number, canvasY: number): Point => {
    return {
        x: (canvasX - offset.value.x) / scale.value,
        y: (canvasY - offset.value.y) / scale.value
    }
}

// 坐标转换：图片原始坐标 -> 画布坐标
const imageToCanvas = (imageX: number, imageY: number): Point => {
    return {
        x: imageX * scale.value + offset.value.x,
        y: imageY * scale.value + offset.value.y
    }
}

// 保存当前图片的标注
const saveCurrentImageAnnotations = () => {
    if (currentImageIndex.value >= 0 && currentImageIndex.value < imageList.value.length) {
        imageList.value[currentImageIndex.value].annotations = [...annotations.value]
    }
}

// 保存标签
const saveLabel = () => {
    if (selectedId.value) {
        const annotation = annotations.value.find(a => a.id === selectedId.value)
        if (annotation) {
            annotation.label = labelText.value || undefined
            // 保存当前图片的标注
            saveCurrentImageAnnotations()
        }
    }
    editingLabel.value = false
    redrawCanvas()
}

// 开始编辑标签
const startEditingLabel = (e: MouseEvent, annotation: Annotation) => {
    e.stopPropagation()

    if (annotation.type === 'rect') {
        const rect = annotation as RectAnnotation
        const canvasRect = imageToCanvas(rect.x, rect.y)
        labelEditorStyle.value = {
            top: `${canvasRect.y - 30}px`,
            left: `${canvasRect.x}px`
        }
    } else if (annotation.type === 'polygon' || annotation.type === 'polyline') {
        const polygon = annotation as PolygonAnnotation | PolylineAnnotation
        if (polygon.points.length > 0) {
            const canvasPoint = imageToCanvas(polygon.points[0].x, polygon.points[0].y)
            labelEditorStyle.value = {
                top: `${canvasPoint.y - 30}px`,
                left: `${canvasPoint.x}px`
            }
        }
    } else if (annotation.type === 'point') {
        const point = annotation as PointAnnotation
        const canvasPoint = imageToCanvas(point.x, point.y)
        labelEditorStyle.value = {
            top: `${canvasPoint.y - 30}px`,
            left: `${canvasPoint.x}px`
        }
    }

    labelText.value = annotation.label || ''
    editingLabel.value = true

    nextTick(() => {
        if (labelInputRef.value) {
            labelInputRef.value.focus()
            labelInputRef.value.select()
        }
    })
}

// 编辑选中标注的标签
const editSelectedAnnotationLabel = () => {
    if (!selectedAnnotation.value) return

    // 模拟点击事件来启动标签编辑
    const mockEvent = new MouseEvent('mousedown', {
        clientX: 100,
        clientY: 100
    }) as MouseEvent
    startEditingLabel(mockEvent, selectedAnnotation.value)
}

// 编辑选中标注的属性
const editSelectedAnnotationProperties = () => {
    if (!selectedAnnotation.value) return

    // 初始化属性编辑器
    currentProperties.value = { ...selectedAnnotation.value.properties || {} }
    const properties = selectedAnnotation.value.properties || {}
    propertyKeys.value = {}
    for (const key in properties) {
        propertyKeys.value[key] = key
    }

    // 设置属性编辑器位置
    const canvas = canvasRef.value
    if (canvas) {
        propertiesEditorStyle.value = {
            top: '50px',
            left: `${canvas.width / 2 - 150}px`
        }
    }

    editingProperties.value = true
}

// 关闭属性编辑器
const closePropertiesEditor = () => {
    editingProperties.value = false
    currentProperties.value = {}
    propertyKeys.value = {}
}

// 保存属性
const saveProperties = () => {
    if (selectedId.value) {
        const annotation = annotations.value.find(a => a.id === selectedId.value)
        if (annotation) {
            // 过滤掉空属性名或属性值
            const filteredProperties: Record<string, string> = {}
            Object.keys(currentProperties.value).forEach(key => {
                const value = currentProperties.value[key]
                if (key.trim() && value.trim()) {
                    filteredProperties[key.trim()] = value.trim()
                }
            })

            annotation.properties = Object.keys(filteredProperties).length > 0 ? filteredProperties : undefined
            // 保存当前图片的标注
            saveCurrentImageAnnotations()
        }
    }
    closePropertiesEditor()
    redrawCanvas()
}

// 添加属性
const addProperty = () => {
    const newKey = `property_${Object.keys(currentProperties.value).length + 1}`
    currentProperties.value[newKey] = ''
    propertyKeys.value[newKey] = newKey
}

// 删除属性
const removeProperty = (key: string) => {
    delete currentProperties.value[key]
    delete propertyKeys.value[key]
}

// 更新属性键
const updatePropertyKey = (oldKey: string, newKey: string) => {
    if (oldKey === newKey) return

    const value = currentProperties.value[oldKey]
    delete currentProperties.value[oldKey]
    delete propertyKeys.value[oldKey]

    if (newKey.trim()) {
        currentProperties.value[newKey] = value
        propertyKeys.value[newKey] = newKey
    }
}

// 更新属性值
const updatePropertyValue = (key: string, value: string) => {
    currentProperties.value[key] = value
}

// 获取多边形/折线中心点
const getPolygonCenter = (points: Point[]): Point => {
    const sum = points.reduce((acc, point) => ({
        x: acc.x + point.x,
        y: acc.y + point.y
    }), { x: 0, y: 0 })

    return {
        x: sum.x / points.length,
        y: sum.y / points.length
    }
}

// 添加点到多边形/折线
const addPointToPolygon = (e: MouseEvent, annotation: PolygonAnnotation | PolylineAnnotation) => {
    e.stopPropagation()

    const canvas = canvasRef.value
    if (!canvas) return

    const rect = canvas.getBoundingClientRect()
    const canvasX = e.clientX - rect.left
    const canvasY = e.clientY - rect.top

    // 限制在图片范围内
    const clampedPoint = clampPointToImage({ x: canvasX, y: canvasY })
    const imagePoint = canvasToImage(clampedPoint.x, clampedPoint.y)

    // 找到最近的边
    let minDistance = Infinity
    let insertIndex = 0

    for (let i = 0; i < annotation.points.length - (annotation.type === 'polyline' ? 1 : 0); i++) {
        const p1 = annotation.points[i]
        const p2 = annotation.points[(i + 1) % annotation.points.length]

        const distance = distanceToLineSegment(
            imagePoint.x, imagePoint.y,
            p1.x, p1.y,
            p2.x, p2.y
        )

        if (distance < minDistance) {
            minDistance = distance
            insertIndex = i + 1
        }
    }

    // 插入新点
    annotation.points.splice(insertIndex, 0, imagePoint)

    // 保存当前图片的标注
    saveCurrentImageAnnotations()

    pointOperationHint.value = '已添加新点'
    setTimeout(() => {
        pointOperationHint.value = ''
    }, 2000)

    redrawCanvas()
}

// 删除多边形/折线的点
const deletePointFromPolygon = (e: MouseEvent, annotation: PolygonAnnotation | PolylineAnnotation, index: number) => {
    e.stopPropagation()

    // 至少保留3个点（多边形）或2个点（折线）
    const minPoints = annotation.type === 'polygon' ? 3 : 2

    if (annotation.points.length <= minPoints) {
        pointOperationHint.value = `至少需要保留${minPoints}个点`
        setTimeout(() => {
            pointOperationHint.value = ''
        }, 2000)
        return
    }

    annotation.points.splice(index, 1)

    // 保存当前图片的标注
    saveCurrentImageAnnotations()

    pointOperationHint.value = '已删除点'
    setTimeout(() => {
        pointOperationHint.value = ''
    }, 2000)

    redrawCanvas()
}

// 根据标签选中标注
const selectAnnotationByLabel = (label: string) => {
    // 找到所有具有该标签的标注
    const labeledAnnotations = annotations.value.filter(a => a.label === label)

    if (labeledAnnotations.length > 0) {
        // 选中第一个具有该标签的标注
        selectedId.value = labeledAnnotations[0].id
        redrawCanvas()
    }
}

// 判断标签是否处于活动状态（是否有对应标注被选中）
const isLabelActive = (label: string): boolean => {
    if (!selectedId.value) return false

    const selectedAnnotation = annotations.value.find(a => a.id === selectedId.value)
    return selectedAnnotation?.label === label
}

// 获取标签对应的标注数量
const getLabelCount = (label: string): number => {
    let count = 0
    imageList.value.forEach(imageData => {
        count += imageData.annotations.filter(a => a.label === label).length
    })
    return count
}

// 获取标注类型名称
const getAnnotationTypeName = (type: AnnotationType): string => {
    const typeMap = {
        rect: '矩形',
        polygon: '多边形',
        polyline: '折线',
        point: '点'
    }
    return typeMap[type]
}

// 监听选中标注的变化，更新标签列表的活动状态
watch(selectedId, (newId) => {
    // 当选中标注变化时，重绘以更新标签列表的活动状态
    redrawCanvas()
})

// 鼠标事件处理
const handleMouseDown = (e: MouseEvent) => {
    const canvas = canvasRef.value
    if (!canvas) return

    const rect = canvas.getBoundingClientRect()
    let canvasX = e.clientX - rect.left
    let canvasY = e.clientY - rect.top

    // 对于选择模式，不限制鼠标位置，以便能选中贴边的标注
    if (mode.value !== 'select') {
        // 限制在图片范围内（仅用于绘制新标注）
        const clampedPoint = clampPointToImage({ x: canvasX, y: canvasY })
        canvasX = clampedPoint.x
        canvasY = clampedPoint.y
    }

    const imagePoint = canvasToImage(canvasX, canvasY)

    startPoint.value = { x: canvasX, y: canvasY }
    lastMousePos.value = { x: canvasX, y: canvasY }

    if (mode.value === 'select') {
        // 检查是否点击了标注（不限制鼠标位置）
        const clickedAnnotation = findAnnotationAtPoint(canvasX, canvasY)

        if (clickedAnnotation) {
            selectedId.value = clickedAnnotation.id

            // 检查是否点击了标签区域
            if (clickedAnnotation.label) {
                const labelBounds = getLabelBounds(clickedAnnotation)
                if (labelBounds &&
                    canvasX >= labelBounds.left && canvasX <= labelBounds.right &&
                    canvasY >= labelBounds.top && canvasY <= labelBounds.bottom) {
                    startEditingLabel(e, clickedAnnotation)
                    return
                }
            }

            // 检查是否点击了控制点或顶点
            if (clickedAnnotation.type === 'rect') {
                const rectAnnotation = clickedAnnotation as RectAnnotation
                const canvasRect = {
                    x: rectAnnotation.x * scale.value + offset.value.x,
                    y: rectAnnotation.y * scale.value + offset.value.y,
                    width: rectAnnotation.width * scale.value,
                    height: rectAnnotation.height * scale.value
                }

                const handles = [
                    { x: canvasRect.x, y: canvasRect.y, type: 'resize', corner: 'tl' },
                    { x: canvasRect.x + canvasRect.width, y: canvasRect.y, type: 'resize', corner: 'tr' },
                    { x: canvasRect.x, y: canvasRect.y + canvasRect.height, type: 'resize', corner: 'bl' },
                    { x: canvasRect.x + canvasRect.width, y: canvasRect.y + canvasRect.height, type: 'resize', corner: 'br' }
                ]

                for (let i = 0; i < handles.length; i++) {
                    const handle = handles[i]
                    const distance = Math.sqrt(Math.pow(canvasX - handle.x, 2) + Math.pow(canvasY - handle.y, 2))
                    if (distance <= 8) {
                        isDragging.value = true
                        dragType.value = 'resize'
                        dragIndex.value = i
                        dragOffset.value = { x: canvasX - handle.x, y: canvasY - handle.y }
                        return
                    }
                }

                // 如果没有点击控制点，则移动整个矩形
                isDragging.value = true
                dragType.value = 'move'
                dragOffset.value = {
                    x: canvasX - canvasRect.x,
                    y: canvasY - canvasRect.y
                }
            } else if (clickedAnnotation.type === 'polygon' || clickedAnnotation.type === 'polyline') {
                const polygonAnnotation = clickedAnnotation as PolygonAnnotation | PolylineAnnotation

                // 检查是否点击了顶点
                for (let i = 0; i < polygonAnnotation.points.length; i++) {
                    const canvasPoint = imageToCanvas(polygonAnnotation.points[i].x, polygonAnnotation.points[i].y)
                    const distance = Math.sqrt(Math.pow(canvasX - canvasPoint.x, 2) + Math.pow(canvasY - canvasPoint.y, 2))
                    if (distance <= 8) {
                        isDragging.value = true
                        dragType.value = 'vertex'
                        dragIndex.value = i
                        return
                    }
                }

                // 检查是否点击了边（用于添加点）
                if (e.shiftKey) {
                    addPointToPolygon(e, polygonAnnotation)
                    return
                }

                // 如果没有点击顶点，则移动整个多边形
                isDragging.value = true
                dragType.value = 'move'

                // 保存多边形初始位置
                polygonDragStartPoints.value = [...polygonAnnotation.points]

                // 计算鼠标相对于多边形中心的偏移
                const centerPoint = getPolygonCenter(polygonAnnotation.points)
                const centerCanvasPoint = imageToCanvas(centerPoint.x, centerPoint.y)
                dragOffset.value = {
                    x: canvasX - centerCanvasPoint.x,
                    y: canvasY - centerCanvasPoint.y
                }
            } else if (clickedAnnotation.type === 'point') {
                const pointAnnotation = clickedAnnotation as PointAnnotation
                const canvasPoint = imageToCanvas(pointAnnotation.x, pointAnnotation.y)
                isDragging.value = true
                dragType.value = 'move'
                dragOffset.value = {
                    x: canvasX - canvasPoint.x,
                    y: canvasY - canvasPoint.y
                }
            }
        } else {
            // 没有点击标注，开始拖动图片
            selectedId.value = null
            isDraggingImage.value = true
            dragType.value = 'image'
        }

        redrawCanvas()
    } else if (mode.value === 'rect') {
        isDrawing.value = true
        tempAnnotation.value = {
            id: generateId(),
            type: 'rect',
            x: imagePoint.x,
            y: imagePoint.y,
            width: 0,
            height: 0,
            color: getRandomColor(),
            label: '未命名',
            properties: {}
        }
    } else if (mode.value === 'polygon' || mode.value === 'polyline') {
        // 添加点（使用图片原始坐标）
        currentPoints.value.push(imagePoint)
        redrawCanvas()
    } else if (mode.value === 'point') {
        const newPoint: PointAnnotation = {
            id: generateId(),
            type: 'point',
            x: imagePoint.x,
            y: imagePoint.y,
            color: getRandomColor(),
            label: '未命名',
            properties: {}
        }
        annotations.value.push(newPoint)
        // 保存当前图片的标注
        saveCurrentImageAnnotations()
        redrawCanvas()
    }
}

// 右键点击事件（删除点）
const handleRightClick = (e: MouseEvent) => {
    e.preventDefault()

    if (mode.value !== 'select' || !selectedId.value) return

    const canvas = canvasRef.value
    if (!canvas) return

    const rect = canvas.getBoundingClientRect()
    const canvasX = e.clientX - rect.left
    const canvasY = e.clientY - rect.top

    const annotation = annotations.value.find(a => a.id === selectedId.value)
    if (!annotation || (annotation.type !== 'polygon' && annotation.type !== 'polyline')) return

    const polygonAnnotation = annotation as PolygonAnnotation | PolylineAnnotation

    // 检查是否点击了顶点
    for (let i = 0; i < polygonAnnotation.points.length; i++) {
        const canvasPoint = imageToCanvas(polygonAnnotation.points[i].x, polygonAnnotation.points[i].y)
        const distance = Math.sqrt(Math.pow(canvasX - canvasPoint.x, 2) + Math.pow(canvasY - canvasPoint.y, 2))
        if (distance <= 8) {
            deletePointFromPolygon(e, polygonAnnotation, i)
            return
        }
    }
}

const handleMouseMove = (e: MouseEvent) => {
    const canvas = canvasRef.value
    if (!canvas) return

    const rect = canvas.getBoundingClientRect()
    let canvasX = e.clientX - rect.left
    let canvasY = e.clientY - rect.top

    // 计算图片坐标点
    const imagePoint = canvasToImage(canvasX, canvasY)

    // 处理图片拖动
    if (isDraggingImage.value && dragType.value === 'image') {
        const dx = canvasX - lastMousePos.value.x
        const dy = canvasY - lastMousePos.value.y
        offset.value.x += dx
        offset.value.y += dy
        lastMousePos.value = { x: canvasX, y: canvasY }
        redrawCanvas()
        return
    }

    // 处理标注拖动
    if (isDragging.value && selectedId.value) {
        const annotation = annotations.value.find(a => a.id === selectedId.value)
        if (!annotation) return

        if (annotation.type === 'rect') {
            const rectAnnotation = annotation as RectAnnotation

            if (dragType.value === 'move') {
                // 移动整个矩形
                const newCanvasX = canvasX - dragOffset.value.x
                const newCanvasY = canvasY - dragOffset.value.y
                const newImagePoint = canvasToImage(newCanvasX, newCanvasY)

                // 直接更新矩形位置
                rectAnnotation.x = newImagePoint.x
                rectAnnotation.y = newImagePoint.y

                // 应用严格的边界限制
                const clampedRect = clampRectToImageStrict(rectAnnotation)
                rectAnnotation.x = clampedRect.x
                rectAnnotation.y = clampedRect.y
                rectAnnotation.width = clampedRect.width
                rectAnnotation.height = clampedRect.height
            } else if (dragType.value === 'resize') {
                // 调整矩形大小
                const corners = ['tl', 'tr', 'bl', 'br']
                const corner = corners[dragIndex.value]

                // 创建临时矩形
                const tempRect = { ...rectAnnotation }

                if (corner === 'tl') {
                    tempRect.width += tempRect.x - imagePoint.x
                    tempRect.height += tempRect.y - imagePoint.y
                    tempRect.x = imagePoint.x
                    tempRect.y = imagePoint.y
                } else if (corner === 'tr') {
                    tempRect.width = imagePoint.x - tempRect.x
                    tempRect.height += tempRect.y - imagePoint.y
                    tempRect.y = imagePoint.y
                } else if (corner === 'bl') {
                    tempRect.width += tempRect.x - imagePoint.x
                    tempRect.height = imagePoint.y - tempRect.y
                    tempRect.x = imagePoint.x
                } else if (corner === 'br') {
                    tempRect.width = imagePoint.x - tempRect.x
                    tempRect.height = imagePoint.y - tempRect.y
                }

                // 确保宽高为正
                if (tempRect.width < 0) {
                    tempRect.x += tempRect.width
                    tempRect.width = Math.abs(tempRect.width)
                }
                if (tempRect.height < 0) {
                    tempRect.y += tempRect.height
                    tempRect.height = Math.abs(tempRect.height)
                }

                // 应用严格的边界限制
                const clampedRect = clampRectToImageStrict(tempRect)

                // 更新矩形
                rectAnnotation.x = clampedRect.x
                rectAnnotation.y = clampedRect.y
                rectAnnotation.width = clampedRect.width
                rectAnnotation.height = clampedRect.height
            }
        } else if (annotation.type === 'polygon' || annotation.type === 'polyline') {
            const polygonAnnotation = annotation as PolygonAnnotation | PolylineAnnotation

            if (dragType.value === 'move') {
                // 计算多边形中心应该移动到的位置
                const newCenterCanvasX = canvasX - dragOffset.value.x
                const newCenterCanvasY = canvasY - dragOffset.value.y
                const newCenterImagePoint = canvasToImage(newCenterCanvasX, newCenterCanvasY)

                // 计算原始中心点
                const originalCenter = getPolygonCenter(polygonDragStartPoints.value)

                // 计算偏移量
                const dx = newCenterImagePoint.x - originalCenter.x
                const dy = newCenterImagePoint.y - originalCenter.y

                // 移动整个多边形
                polygonAnnotation.points = polygonDragStartPoints.value.map(point => ({
                    x: point.x + dx,
                    y: point.y + dy
                }))

                // 限制所有点在图片范围内
                const bounds = getImageBounds()
                if (bounds) {
                    polygonAnnotation.points = polygonAnnotation.points.map(point => {
                        const canvasPoint = imageToCanvas(point.x, point.y)
                        const clampedCanvasPoint = clampPointToImage(canvasPoint)
                        return canvasToImage(clampedCanvasPoint.x, clampedCanvasPoint.y)
                    })
                }
            } else if (dragType.value === 'vertex') {
                // 移动顶点
                polygonAnnotation.points[dragIndex.value] = imagePoint

                // 限制点在图片范围内
                const bounds = getImageBounds()
                if (bounds) {
                    const canvasPoint = imageToCanvas(imagePoint.x, imagePoint.y)
                    const clampedCanvasPoint = clampPointToImage(canvasPoint)
                    polygonAnnotation.points[dragIndex.value] = canvasToImage(clampedCanvasPoint.x, clampedCanvasPoint.y)
                }
            }
        } else if (annotation.type === 'point') {
            const pointAnnotation = annotation as PointAnnotation

            if (dragType.value === 'move') {
                // 计算点应该移动到的位置
                const newCanvasX = canvasX - dragOffset.value.x
                const newCanvasY = canvasY - dragOffset.value.y
                const newImagePoint = canvasToImage(newCanvasX, newCanvasY)

                pointAnnotation.x = newImagePoint.x
                pointAnnotation.y = newImagePoint.y

                // 限制点在图片范围内
                const bounds = getImageBounds()
                if (bounds) {
                    const canvasPoint = imageToCanvas(pointAnnotation.x, pointAnnotation.y)
                    const clampedCanvasPoint = clampPointToImage(canvasPoint)
                    pointAnnotation.x = canvasToImage(clampedCanvasPoint.x, clampedCanvasPoint.y).x
                    pointAnnotation.y = canvasToImage(clampedCanvasPoint.x, clampedCanvasPoint.y).y
                }
            }
        }

        // 保存当前图片的标注
        saveCurrentImageAnnotations()

        redrawCanvas()
        return
    }

    // 处理绘制
    if (isDrawing.value && tempAnnotation.value && tempAnnotation.value.type === 'rect') {
        const rectAnnotation = tempAnnotation.value as RectAnnotation
        rectAnnotation.width = imagePoint.x - rectAnnotation.x
        rectAnnotation.height = imagePoint.y - rectAnnotation.y

        // 确保宽高为正
        if (rectAnnotation.width < 0) {
            rectAnnotation.x += rectAnnotation.width
            rectAnnotation.width = Math.abs(rectAnnotation.width)
        }
        if (rectAnnotation.height < 0) {
            rectAnnotation.y += rectAnnotation.height
            rectAnnotation.height = Math.abs(rectAnnotation.height)
        }

        // 限制在图片范围内
        const clampedRect = clampRectToImageStrict(rectAnnotation)
        rectAnnotation.x = clampedRect.x
        rectAnnotation.y = clampedRect.y
        rectAnnotation.width = clampedRect.width
        rectAnnotation.height = clampedRect.height

        redrawCanvas()
    } else if ((mode.value === 'polygon' || mode.value === 'polyline') && currentPoints.value.length > 0) {
        // 限制当前鼠标位置在图片范围内
        const clampedCanvasPoint = clampPointToImage({ x: canvasX, y: canvasY })

        // 绘制临时多边形/折线
        redrawCanvas()
        const ctx = canvas.getContext('2d')
        if (ctx) {
            ctx.strokeStyle = '#ff0000'
            ctx.lineWidth = 2
            ctx.beginPath()

            // 将多边形/折线点转换为画布坐标
            const firstCanvasPoint = imageToCanvas(currentPoints.value[0].x, currentPoints.value[0].y)
            ctx.moveTo(firstCanvasPoint.x, firstCanvasPoint.y)

            for (let i = 1; i < currentPoints.value.length; i++) {
                const canvasPoint = imageToCanvas(currentPoints.value[i].x, currentPoints.value[i].y)
                ctx.lineTo(canvasPoint.x, canvasPoint.y)
            }

            ctx.lineTo(clampedCanvasPoint.x, clampedCanvasPoint.y)

            // 如果是多边形，则闭合
            if (mode.value === 'polygon') {
                ctx.closePath()
            }

            ctx.stroke()

            // 绘制已确定的点
            ctx.fillStyle = '#ff0000'
            currentPoints.value.forEach(point => {
                const canvasPoint = imageToCanvas(point.x, point.y)
                ctx.beginPath()
                ctx.arc(canvasPoint.x, canvasPoint.y, 3, 0, Math.PI * 2)
                ctx.fill()
            })
        }
    }
}

const handleMouseUp = (e: MouseEvent) => {
    if (isDrawing.value && tempAnnotation.value && tempAnnotation.value.type === 'rect') {
        const rectAnnotation = tempAnnotation.value as RectAnnotation

        // 确保宽高为正
        if (rectAnnotation.width < 0) {
            rectAnnotation.x += rectAnnotation.width
            rectAnnotation.width = Math.abs(rectAnnotation.width)
        }
        if (rectAnnotation.height < 0) {
            rectAnnotation.y += rectAnnotation.height
            rectAnnotation.height = Math.abs(rectAnnotation.height)
        }

        // 只保存有效矩形
        if (rectAnnotation.width > 5 && rectAnnotation.height > 5) {
            annotations.value.push(rectAnnotation)
            // 保存当前图片的标注
            saveCurrentImageAnnotations()
        }

        isDrawing.value = false
        tempAnnotation.value = null
        redrawCanvas()
    }

    // 结束拖动
    isDragging.value = false
    isDraggingImage.value = false
    dragType.value = null
    polygonDragStartPoints.value = []
}

const handleDoubleClick = (e: MouseEvent) => {
    if ((mode.value === 'polygon' || mode.value === 'polyline') && currentPoints.value.length >= 2) {
        const minPoints = mode.value === 'polygon' ? 3 : 2

        if (currentPoints.value.length < minPoints) {
            pointOperationHint.value = `${mode.value === 'polygon' ? '多边形' : '折线'}至少需要${minPoints}个点`
            setTimeout(() => {
                pointOperationHint.value = ''
            }, 2000)
            return
        }

        const newAnnotation: PolygonAnnotation | PolylineAnnotation = {
            id: generateId(),
            type: mode.value,
            points: [...currentPoints.value],
            color: getRandomColor(),
            label: '未命名',
            properties: {}
        }
        annotations.value.push(newAnnotation)
        // 保存当前图片的标注
        saveCurrentImageAnnotations()
        currentPoints.value = []
        redrawCanvas()
    }
}

// 处理鼠标滚轮（缩放）
const handleWheel = (e: WheelEvent) => {
    e.preventDefault()

    const canvas = canvasRef.value
    if (!canvas || !backgroundImage.value) return

    const rect = canvas.getBoundingClientRect()
    const canvasX = e.clientX - rect.left
    const canvasY = e.clientY - rect.top

    // 计算鼠标在图片上的位置（原始坐标）
    const imagePoint = canvasToImage(canvasX, canvasY)

    // 计算缩放因子
    const zoomIntensity = 0.1
    const wheel = e.deltaY < 0 ? 1 : -1
    const newScale = Math.max(0.1, Math.min(5, scale.value + wheel * zoomIntensity))

    // 计算缩放后的偏移量，使鼠标位置保持不变
    const newOffsetX = canvasX - imagePoint.x * newScale
    const newOffsetY = canvasY - imagePoint.y * newScale

    scale.value = newScale
    offset.value = { x: newOffsetX, y: newOffsetY }

    redrawCanvas()
}

// 获取标签边界
const getLabelBounds = (annotation: Annotation): { left: number, top: number, right: number, bottom: number } | null => {
    if (!annotation.label) return null

    const ctx = canvasRef.value?.getContext('2d')
    if (!ctx) return null

    ctx.font = '14px Arial'
    const textWidth = ctx.measureText(annotation.label).width
    const textHeight = 20

    if (annotation.type === 'rect') {
        const rect = annotation as RectAnnotation
        const canvasPoint = imageToCanvas(rect.x, rect.y)
        return {
            left: canvasPoint.x,
            top: canvasPoint.y - textHeight - 5,
            right: canvasPoint.x + textWidth + 10,
            bottom: canvasPoint.y - 5
        }
    } else if (annotation.type === 'polygon' || annotation.type === 'polyline') {
        const polygon = annotation as PolygonAnnotation | PolylineAnnotation
        if (polygon.points.length > 0) {
            const canvasPoint = imageToCanvas(polygon.points[0].x, polygon.points[0].y)
            return {
                left: canvasPoint.x,
                top: canvasPoint.y - textHeight - 5,
                right: canvasPoint.x + textWidth + 10,
                bottom: canvasPoint.y - 5
            }
        }
    } else if (annotation.type === 'point') {
        const point = annotation as PointAnnotation
        const canvasPoint = imageToCanvas(point.x, point.y)
        return {
            left: canvasPoint.x,
            top: canvasPoint.y - textHeight - 5,
            right: canvasPoint.x + textWidth + 10,
            bottom: canvasPoint.y - 5
        }
    }

    return null
}

// 查找点击位置的标注（使用画布坐标）
const findAnnotationAtPoint = (canvasX: number, canvasY: number): Annotation | null => {
    // 从后往前查找（最后绘制的在最上层）
    for (let i = annotations.value.length - 1; i >= 0; i--) {
        const annotation = annotations.value[i]

        if (annotation.type === 'rect') {
            const rect = annotation as RectAnnotation
            const canvasRect = {
                x: rect.x * scale.value + offset.value.x,
                y: rect.y * scale.value + offset.value.y,
                width: rect.width * scale.value,
                height: rect.height * scale.value
            }

            // 扩大检测范围，使贴边的标注更容易选中
            const expandedRect = {
                x: canvasRect.x - 5,
                y: canvasRect.y - 5,
                width: canvasRect.width + 10,
                height: canvasRect.height + 10
            }

            if (canvasX >= expandedRect.x && canvasX <= expandedRect.x + expandedRect.width &&
                canvasY >= expandedRect.y && canvasY <= expandedRect.y + expandedRect.height) {
                return annotation
            }
        } else if (annotation.type === 'polygon') {
            const polygon = annotation as PolygonAnnotation

            // 将多边形点转换为画布坐标
            const canvasPoints = polygon.points.map(point =>
                imageToCanvas(point.x, point.y)
            )

            if (isPointInPolygonCanvas(canvasX, canvasY, canvasPoints)) {
                return annotation
            }

            // 如果点不在多边形内，检查是否靠近任何边
            for (let i = 0; i < canvasPoints.length; i++) {
                const p1 = canvasPoints[i]
                const p2 = canvasPoints[(i + 1) % canvasPoints.length]

                if (distanceToLineSegment(canvasX, canvasY, p1.x, p1.y, p2.x, p2.y) < 8) {
                    return annotation
                }
            }
        } else if (annotation.type === 'polyline') {
            const polyline = annotation as PolylineAnnotation

            // 将折线点转换为画布坐标
            const canvasPoints = polyline.points.map(point =>
                imageToCanvas(point.x, point.y)
            )

            // 检查是否靠近任何边
            for (let i = 0; i < canvasPoints.length - 1; i++) {
                const p1 = canvasPoints[i]
                const p2 = canvasPoints[i + 1]

                if (distanceToLineSegment(canvasX, canvasY, p1.x, p1.y, p2.x, p2.y) < 8) {
                    return annotation
                }
            }

            // 检查是否靠近任何顶点
            for (let i = 0; i < canvasPoints.length; i++) {
                const p = canvasPoints[i]
                const distance = Math.sqrt(Math.pow(canvasX - p.x, 2) + Math.pow(canvasY - p.y, 2))
                if (distance <= 8) {
                    return annotation
                }
            }
        } else if (annotation.type === 'point') {
            const point = annotation as PointAnnotation
            const canvasPoint = imageToCanvas(point.x, point.y)
            const distance = Math.sqrt(Math.pow(canvasX - canvasPoint.x, 2) + Math.pow(canvasY - canvasPoint.y, 2))

            // 扩大检测范围
            if (distance <= 10) {
                return annotation
            }
        }
    }
    return null
}

// 判断点是否在多边形内（使用画布坐标）
const isPointInPolygonCanvas = (x: number, y: number, points: Point[]): boolean => {
    let inside = false
    for (let i = 0, j = points.length - 1; i < points.length; j = i++) {
        const xi = points[i].x, yi = points[i].y
        const xj = points[j].x, yj = points[j].y

        const intersect = ((yi > y) !== (yj > y))
            && (x < (xj - xi) * (y - yi) / (yj - yi) + xi)
        if (intersect) inside = !inside
    }
    return inside
}

// 计算点到线段的距离
const distanceToLineSegment = (x: number, y: number, x1: number, y1: number, x2: number, y2: number): number => {
    const A = x - x1
    const B = y - y1
    const C = x2 - x1
    const D = y2 - y1

    const dot = A * C + B * D
    const lenSq = C * C + D * D
    let param = -1

    if (lenSq !== 0) {
        param = dot / lenSq
    }

    let xx, yy

    if (param < 0) {
        xx = x1
        yy = y1
    } else if (param > 1) {
        xx = x2
        yy = y2
    } else {
        xx = x1 + param * C
        yy = y1 + param * D
    }

    const dx = x - xx
    const dy = y - yy

    return Math.sqrt(dx * dx + dy * dy)
}

// 删除选中的标注
const deleteSelected = () => {
    if (selectedId.value) {
        annotations.value = annotations.value.filter(anno => anno.id !== selectedId.value)
        // 保存当前图片的标注
        saveCurrentImageAnnotations()
        selectedId.value = null
        editingLabel.value = false
        editingProperties.value = false
        redrawCanvas()
    }
}

// 导出标注数据
const exportAnnotations = () => {
    // 保存当前图片的标注
    saveCurrentImageAnnotations()

    // 导出所有图片的标注数据
    const data = {
        images: imageList.value,
        timestamp: new Date().toISOString()
    }
    const json = JSON.stringify(data, null, 2)
    const blob = new Blob([json], { type: 'application/json' })
    const url = URL.createObjectURL(blob)

    const a = document.createElement('a')
    a.href = url
    a.download = `annotations_${Date.now()}.json`
    document.body.appendChild(a)
    a.click()
    document.body.removeChild(a)
    URL.revokeObjectURL(url)
}

// 重绘画布
const redrawCanvas = () => {
    const canvas = canvasRef.value
    if (!canvas) return

    const ctx = canvas.getContext('2d')
    if (!ctx) return

    // 清空画布
    ctx.clearRect(0, 0, canvas.width, canvas.height)

    // 绘制背景
    ctx.fillStyle = '#f0f0f0'
    ctx.fillRect(0, 0, canvas.width, canvas.height)

    // 绘制图片
    if (backgroundImage.value) {
        ctx.save()
        ctx.translate(offset.value.x, offset.value.y)
        ctx.scale(scale.value, scale.value)
        ctx.drawImage(backgroundImage.value, 0, 0)
        ctx.restore()
    } else {
        ctx.fillStyle = '#333'
        ctx.font = '20px Arial'
        ctx.textAlign = 'center'
        ctx.fillText('请加载图片开始标注', canvas.width / 2, canvas.height / 2)
    }

    // 绘制所有标注
    annotations.value.forEach(annotation => {
        drawAnnotation(ctx, annotation)
    })

    // 绘制临时标注
    if (tempAnnotation.value) {
        drawAnnotation(ctx, tempAnnotation.value)
    }

    // 绘制当前多边形/折线点
    if ((mode.value === 'polygon' || mode.value === 'polyline') && currentPoints.value.length > 0) {
        ctx.fillStyle = '#ff0000'
        currentPoints.value.forEach(point => {
            const canvasPoint = imageToCanvas(point.x, point.y)
            ctx.beginPath()
            ctx.arc(canvasPoint.x, canvasPoint.y, 3, 0, Math.PI * 2)
            ctx.fill()
        })
    }
}

// 绘制单个标注
const drawAnnotation = (ctx: CanvasRenderingContext2D, annotation: Annotation) => {
    const isSelected = annotation.id === selectedId.value

    if (annotation.type === 'rect') {
        const rect = annotation as RectAnnotation
        const canvasRect = {
            x: rect.x * scale.value + offset.value.x,
            y: rect.y * scale.value + offset.value.y,
            width: rect.width * scale.value,
            height: rect.height * scale.value
        }

        ctx.strokeStyle = isSelected ? '#ff0000' : annotation.color
        ctx.lineWidth = isSelected ? 3 : 2
        ctx.strokeRect(canvasRect.x, canvasRect.y, canvasRect.width, canvasRect.height)

        // 绘制标签
        if (annotation.label) {
            ctx.fillStyle = isSelected ? '#ff0000' : annotation.color
            ctx.font = '14px Arial'
            ctx.textAlign = 'left'
            ctx.textBaseline = 'bottom'
            ctx.fillText(annotation.label, canvasRect.x, canvasRect.y - 5)
        }

        if (isSelected) {
            // 绘制控制点
            ctx.fillStyle = '#ff0000'
            const handles = [
                { x: canvasRect.x, y: canvasRect.y },
                { x: canvasRect.x + canvasRect.width, y: canvasRect.y },
                { x: canvasRect.x, y: canvasRect.y + canvasRect.height },
                { x: canvasRect.x + canvasRect.width, y: canvasRect.y + canvasRect.height }
            ]
            handles.forEach(handle => {
                ctx.fillRect(handle.x - 4, handle.y - 4, 8, 8)
            })
        }
    } else if (annotation.type === 'polygon' || annotation.type === 'polyline') {
        const polygon = annotation as PolygonAnnotation | PolylineAnnotation
        ctx.strokeStyle = isSelected ? '#ff0000' : annotation.color
        ctx.lineWidth = isSelected ? 3 : 2
        ctx.beginPath()

        // 将多边形/折线点转换为画布坐标
        const firstCanvasPoint = imageToCanvas(polygon.points[0].x, polygon.points[0].y)
        ctx.moveTo(firstCanvasPoint.x, firstCanvasPoint.y)

        for (let i = 1; i < polygon.points.length; i++) {
            const canvasPoint = imageToCanvas(polygon.points[i].x, polygon.points[i].y)
            ctx.lineTo(canvasPoint.x, canvasPoint.y)
        }

        // 如果是多边形，则闭合
        if (annotation.type === 'polygon') {
            ctx.closePath()
        }

        ctx.stroke()

        // 绘制标签
        if (annotation.label && polygon.points.length > 0) {
            ctx.fillStyle = isSelected ? '#ff0000' : annotation.color
            ctx.font = '14px Arial'
            ctx.textAlign = 'left'
            ctx.textBaseline = 'bottom'
            ctx.fillText(annotation.label, firstCanvasPoint.x, firstCanvasPoint.y - 5)
        }

        if (isSelected) {
            // 绘制顶点控制点
            ctx.fillStyle = '#ff0000'
            polygon.points.forEach(point => {
                const canvasPoint = imageToCanvas(point.x, point.y)
                ctx.beginPath()
                ctx.arc(canvasPoint.x, canvasPoint.y, 4, 0, Math.PI * 2)
                ctx.fill()
            })

            // 如果按住Shift键，显示添加点的提示
            if (pointOperationHint.value === '') {
                ctx.fillStyle = '#333'
                ctx.font = '12px Arial'
                ctx.textAlign = 'center'
                ctx.fillText('按住Shift点击边添加点，右键点击顶点删除点', canvasRef.value.width / 2, 30)
            }
        }
    } else if (annotation.type === 'point') {
        const point = annotation as PointAnnotation
        const canvasPoint = imageToCanvas(point.x, point.y)

        ctx.fillStyle = isSelected ? '#ff0000' : annotation.color
        ctx.beginPath()
        ctx.arc(canvasPoint.x, canvasPoint.y, isSelected ? 6 : 4, 0, Math.PI * 2)
        ctx.fill()

        // 绘制标签
        if (annotation.label) {
            ctx.fillStyle = isSelected ? '#ff0000' : annotation.color
            ctx.font = '14px Arial'
            ctx.textAlign = 'left'
            ctx.textBaseline = 'bottom'
            ctx.fillText(annotation.label, canvasPoint.x, canvasPoint.y - 10)
        }

        if (isSelected) {
            ctx.strokeStyle = '#ffffff'
            ctx.lineWidth = 2
            ctx.stroke()
        }
    }
}

// 工具函数
const generateId = (): string => {
    return Math.random().toString(36).substring(2, 9)
}

const getRandomColor = (): string => {
    const colors = ['#FF5733', '#33FF57', '#3357FF', '#F333FF', '#FF33A1', '#33FFF6']
    return colors[Math.floor(Math.random() * colors.length)]
}
</script>

<style scoped>
.annotation-tool {
    display: flex;
    flex-direction: column;
    height: 100vh;
    padding: 20px;
    box-sizing: border-box;
    background-color: #f5f5f5;
}

.toolbar {
    display: flex;
    gap: 10px;
    margin-bottom: 20px;
    flex-wrap: wrap;
}

.toolbar button {
    padding: 8px 16px;
    background: #f0f0f0;
    border: 1px solid #ccc;
    border-radius: 4px;
    cursor: pointer;
}

.toolbar button:hover {
    background: #e0e0e0;
}

.toolbar button.active {
    background: #4CAF50;
    color: white;
    border-color: #45a049;
}

.toolbar button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
}

.main-content {
    display: flex;
    flex: 1;
    gap: 20px;
    overflow: hidden;
}

.left-panel {
    width: 220px;
    display: flex;
    flex-direction: column;
    gap: 20px;
}

.image-list,
.label-list {
    background: white;
    border: 1px solid #ccc;
    border-radius: 4px;
    padding: 10px;
    overflow-y: auto;
    flex: 1;
}

.image-list h3,
.label-list h3 {
    margin-top: 0;
    margin-bottom: 10px;
    font-size: 16px;
    color: #333;
}

.image-items,
.label-items {
    display: flex;
    flex-direction: column;
    gap: 5px;
}

.image-item {
    padding: 8px;
    background: #f5f5f5;
    border-radius: 4px;
    cursor: pointer;
    display: flex;
    align-items: center;
    transition: background-color 0.2s;
}

.image-item:hover {
    background: #e0e0e0;
}

.image-item.active {
    background: #4CAF50;
    color: white;
}

.image-thumbnail {
    width: 40px;
    height: 40px;
    margin-right: 10px;
    border-radius: 4px;
    overflow: hidden;
    flex-shrink: 0;
}

.image-thumbnail img {
    width: 100%;
    height: 100%;
    object-fit: cover;
}

.image-info {
    flex: 1;
    min-width: 0;
}

.image-name {
    font-weight: bold;
    margin-bottom: 4px;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}

.annotation-count {
    font-size: 12px;
    color: #666;
}

.no-images,
.no-labels {
    padding: 10px;
    text-align: center;
    color: #999;
    font-style: italic;
}

.image-actions {
    display: flex;
    gap: 10px;
    margin-top: 10px;
}

.add-image-btn,
.remove-image-btn {
    flex: 1;
    padding: 6px 10px;
    background: #f0f0f0;
    border: 1px solid #ccc;
    border-radius: 4px;
    cursor: pointer;
    font-size: 12px;
}

.add-image-btn:hover,
.remove-image-btn:hover {
    background: #e0e0e0;
}

.remove-image-btn:disabled {
    opacity: 0.5;
    cursor: not-allowed;
}

.label-item {
    padding: 8px 12px;
    background: #f5f5f5;
    border-radius: 4px;
    cursor: pointer;
    display: flex;
    justify-content: space-between;
    align-items: center;
    transition: background-color 0.2s;
}

.label-item:hover {
    background: #e0e0e0;
}

.label-item.active {
    background: #4CAF50;
    color: white;
}

.label-count {
    font-size: 12px;
    opacity: 0.8;
}

.canvas-container {
    flex: 1;
    border: 1px solid #ccc;
    border-radius: 4px;
    overflow: hidden;
    background: #f9f9f9;
    position: relative;
}

canvas {
    display: block;
    cursor: grab;
    width: 100%;
    height: 100%;
}

canvas:active {
    cursor: grabbing;
}

canvas.select-mode {
    cursor: default;
}

.label-editor {
    position: absolute;
    z-index: 10;
    background: white;
    border: 1px solid #ccc;
    border-radius: 4px;
    padding: 5px;
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
}

.label-editor input {
    border: none;
    outline: none;
    padding: 3px;
    font-size: 14px;
    width: 150px;
}

.properties-editor {
    position: absolute;
    z-index: 10;
    background: white;
    border: 1px solid #ccc;
    border-radius: 4px;
    padding: 10px;
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
    width: 300px;
}

.properties-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 10px;
}

.properties-header h4 {
    margin: 0;
    font-size: 16px;
}

.close-btn {
    background: none;
    border: none;
    font-size: 18px;
    cursor: pointer;
    color: #666;
}

.close-btn:hover {
    color: #000;
}

.properties-content {
    margin-bottom: 15px;
}

.property-item {
    display: flex;
    gap: 5px;
    margin-bottom: 8px;
}

.property-item input {
    flex: 1;
    padding: 5px;
    border: 1px solid #ccc;
    border-radius: 3px;
}

.property-item input:first-child {
    flex: 0.8;
}

.property-item input:last-child {
    flex: 1.2;
}

.remove-property-btn {
    background: #ff4444;
    color: white;
    border: none;
    border-radius: 3px;
    width: 24px;
    height: 24px;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
}

.remove-property-btn:hover {
    background: #cc0000;
}

.add-property-btn {
    width: 100%;
    padding: 6px;
    background: #4CAF50;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
}

.add-property-btn:hover {
    background: #45a049;
}

.properties-footer {
    display: flex;
    gap: 10px;
    justify-content: flex-end;
}

.save-btn,
.cancel-btn {
    padding: 6px 12px;
    border: none;
    border-radius: 4px;
    cursor: pointer;
}

.save-btn {
    background: #4CAF50;
    color: white;
}

.save-btn:hover {
    background: #45a049;
}

.cancel-btn {
    background: #f0f0f0;
    color: #333;
}

.cancel-btn:hover {
    background: #e0e0e0;
}

.hint {
    position: absolute;
    top: 10px;
    left: 50%;
    transform: translateX(-50%);
    background: rgba(0, 0, 0, 0.7);
    color: white;
    padding: 5px 10px;
    border-radius: 4px;
    font-size: 14px;
    z-index: 10;
}

.annotation-properties-panel {
    position: fixed;
    right: 20px;
    top: 100px;
    width: 250px;
    background: white;
    border: 1px solid #ccc;
    border-radius: 4px;
    padding: 15px;
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
    z-index: 5;
}

.annotation-properties-panel h3 {
    margin-top: 0;
    margin-bottom: 15px;
    font-size: 16px;
    color: #333;
}

.annotation-info {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.info-item {
    display: flex;
    flex-direction: column;
    gap: 5px;
}

.info-item .label {
    font-weight: bold;
    color: #666;
    font-size: 14px;
}

.info-item .value {
    font-size: 14px;
    color: #333;
}

.edit-btn {
    align-self: flex-start;
    padding: 4px 8px;
    background: #4CAF50;
    color: white;
    border: none;
    border-radius: 3px;
    cursor: pointer;
    font-size: 12px;
}

.edit-btn:hover {
    background: #45a049;
}

.properties-list {
    margin-top: 5px;
}

.property-display {
    display: flex;
    gap: 5px;
    margin-bottom: 3px;
    font-size: 13px;
}

.property-key {
    font-weight: bold;
    color: #666;
}

.property-value {
    color: #333;
}

.no-properties {
    font-style: italic;
    color: #999;
    font-size: 13px;
}

.status {
    margin-top: 15px;
    font-size: 14px;
    color: #666;
}

.status span {
    margin-right: 20px;
}
</style>