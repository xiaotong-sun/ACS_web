<template>
    <div class="body">
        <div class="top">
            <div id="container"></div>
            <div id="world-cloud"></div>
        </div>
        <div class="bottom">
            <div class="mapspace" id="mapspace"></div>
            <div class="input">
                <el-upload
                        ref="upload"
                        class="upload-demo"
                        action="http://localhost:8080/graph/init"
                        :limit="1"
                        accept=".json"
                        :on-success="graphUploadSuccess"
                >
                    <template #trigger>
                        <el-button type="primary">请选择属性图文件</el-button>
                    </template>
                    <br><br>
                    <template #tip>
                        <div class="el-upload__tip text-red">
                            limit 1 file, new file will cover the old file
                        </div>
                    </template>
                </el-upload>

                <el-form
                        ref="ruleFormRef"
                        :model="ruleForm"
                        status-icon
                        :rules="rules"
                        label-width="120px"
                        class="demo-ruleForm"
                >
                    <el-form-item label="QueryVertex" prop="vertex">
                        <el-input v-model="ruleForm.vertex" placeholder="请输入节点的Label" autocomplete="off"/>
                    </el-form-item>
                    <el-form-item label="K" prop="coreK">
                        <el-input
                                v-model.number="ruleForm.coreK"
                                placeholder="请输入正整数K"
                                autocomplete="off"
                        />
                    </el-form-item>
                    <el-form-item label="Keywords" prop="keywords">
                        <el-input
                                v-model="ruleForm.keywords"
                                placeholder="请输入关键词，关键词之间以 , 分割"
                        />
                    </el-form-item>
                    <el-form-item>
                        <el-button type="primary" @click="submitForm(ruleFormRef)"
                        >提交搜索
                        </el-button
                        >
                        <el-button @click="resetForm(ruleFormRef)">重置</el-button>
                    </el-form-item>
                </el-form>
                <div class="title">Attributed Community Search</div>
            </div>
        </div>
    </div>
</template>

<script setup>
import G6 from '@antv/g6';
import {WordCloud} from '@antv/g2plot';
import {reactive, ref} from "vue";
import axios from "axios";

// ---minimap插件设置---
const minimap = new G6.Minimap({
    container: 'mapspace',
    size: [260, 180],
    hideEdge: true,
});
// ---节点文本提示插件设置---
const tooltip = new G6.Tooltip({
    offsetX: 10,
    offsetY: 0,
    fixToNode: [1, 0.5],
    itemTypes: ['node'],
    getContent: (e) => {
        const outDiv = document.createElement('div');
        outDiv.style.width = "150px";
        outDiv.style.wordBreak = "break-all";
        outDiv.innerHTML = `
        <b>Label:</b><br>${e.item.getModel().id}<br>
        <b>Keywords:</b><br>{${e.item.getModel().keywords}}`;
        return outDiv;
    },
});

function mapNodeSize(nodes, propertyName, visualRange) {
    let minp = 9999999999;
    let maxp = -9999999999;
    nodes.forEach((node) => {
        node[propertyName] = Math.pow(node[propertyName], 1 / 3);
        minp = node[propertyName] < minp ? node[propertyName] : minp;
        maxp = node[propertyName] > maxp ? node[propertyName] : maxp;
    });
    const rangepLength = maxp - minp;
    const rangevLength = visualRange[1] - visualRange[0];
    nodes.forEach((node) => {
        node.size = ((node[propertyName] - minp) / rangepLength) * rangevLength + visualRange[0];
    });
}

function fittingString(str, maxWidth, fontSize) {
    const ellipsis = '...';
    const ellipsisLength = G6.Util.getTextSize(ellipsis, fontSize)[0];
    let currentWidth = 0;
    let res = str;
    const pattern = new RegExp('[\u4E00-\u9FA5]+');
    str.split('').forEach((letter, i) => {
        if (currentWidth > maxWidth - ellipsisLength) return;
        if (pattern.test(letter)) {
            currentWidth += fontSize;
        } else {
            currentWidth += G6.Util.getLetterWidth(letter, fontSize);
        }
        if (currentWidth > maxWidth - ellipsisLength) {
            res = `${str.substring(0, i)}${ellipsis}`;
        }
    });
    return res;
}

const globalFontSize = 12;

// 上传成功回调函数
function graphUploadSuccess(res) {
    const container = document.getElementById('container');
    container.innerHTML = "";
    if (document.getElementById('description') === null) {
        const descriptionDiv = document.createElement('div');
        descriptionDiv.id = 'description';
        container.appendChild(descriptionDiv);
    }
    const descriptionDiv = document.getElementById('description');
    descriptionDiv.innerHTML = `正在渲染大规模数据，请稍后......`;

    const graph = new G6.Graph({
        container: 'container',
        width: 1400,
        height: 728,
        fitCenter: true,
        fitView: true,
        fitViewPadding: [20, 40, 50, 20],
        plugins: [tooltip, minimap],
        defaultNode: {
            size: 5,
            style: {
                fill: '#C6E5FF',
                stroke: '#5B8FF9',
                lineWidth: 0.3,
            },
            labelCfg: {
                style: {
                    fontSize: 3,
                    fill: '#000',
                },
                position: 'right',
                offset: 1,
            },
        },
        defaultEdge: {
            size: 0.1,
            color: '#333',
        },
        layout: {
            type: 'gForce',
            linkDistance: 100,
            nodeStrength: 500,
            preventOverlap: true,
            nodeSpacing: 10,
        },
        modes: {
            default: [
                {
                    type: 'zoom-canvas',
                    enableOptimize: true,
                    optimizeZoom: 0.9,
                },
                {
                    type: 'drag-canvas',
                    enableOptimize: true,
                },
                'click-select',
            ],
        },
        nodeStateStyles: {
            activeByLegend: {
                lineWidth: 2,
                strokeOpacity: 0.5,
                stroke: '#f00',
                fill: '#FFBD32'
            },
            inactiveByLegend: {
                opacity: 0.5,
            },
            selected: {
                lineWidth: 2,
            },
            hover: {
                fill: 'lightsteelblue',
            },
        },
    });

    graph.on('node:mouseenter', (node) => {
        const nodeItem = node.item;
        graph.setItemState(nodeItem, 'hover', true);
    });
    graph.on('node:mouseleave', (node) => {
        const nodeItem = node.item;
        graph.setItemState(nodeItem, 'hover', false);
    });

    const clearStates = () => {
        graph.getNodes().forEach((node) => {
            graph.clearItemStates(node);
        });
        graph.getEdges().forEach((edge) => {
            graph.clearItemStates(edge);
        });
    };

    graph.on('canvas:click', () => {
        clearStates();
    });

// listen to the node click event
    function handleNodeClick(event) {
        const item = event.item;
        graph.focusItem(item, true, {
            easing: 'easeCubic',
            duration: 500,
        });
    }
    graph.on('node:click', handleNodeClick);

    let data = res.data.graphData;

    data.nodes.forEach((node) => {
        // 处理超长文本
        node.label = fittingString(node.label, 100, globalFontSize);

        // 获取节点的度
        node.degree = 0;
        data.edges.forEach((edge) => {
            if (edge.source === node.id || edge.target === node.id) {
                node.degree++;
            }
        });
    });
    let totalDegree = 0;
    data.nodes.forEach((node) => {
        totalDegree += node.degree;
    })
    mapNodeSize(data.nodes, 'degree', [1, 10]);
    graph.data(data);
    graph.render();

    const graphData = graph.save();
    const nodeLen = graphData.nodes.length;
    const edgeLen = graphData.edges.length;
    descriptionDiv.innerHTML = `节点数量：${nodeLen}, 边数量：${edgeLen}, 图元数量：${nodeLen * 2 + edgeLen}, 平均度：${totalDegree / nodeLen}`

    // 词云
    const wordcloudContainer = document.getElementById('world-cloud');
    wordcloudContainer.innerHTML = "";
    const wordCloud = new WordCloud('world-cloud', {
        data: res.data.cloudDatas,
        wordField: 'keyword',
        weightField: 'value',
        color: '#122c6a',
        wordStyle: {
            fontFamily: 'Verdana',
            fontSize: [15, 60],
            rotation: 0,
        },
        random: () => 0.5,
        // 设置交互类型
        interactions: [{type: 'element-active'}],
        state: {
            active: {
                // 这里可以设置 active 时的样式
                style: {
                    lineWidth: 3,
                },
            },
        },
    });
    wordCloud.render();
}

// *****搜索参数*****
const ruleFormRef = ref();
const checkKeywords = (rule, value, callback) => {
    if (value === '') {
        callback(new Error('please input the query Keywords'))
    } else {
        callback();
    }
};
const checkQueryVertex = (rule, value, callback) => {
    if (value === '') {
        callback(new Error('Please input the query Vertex'));
    } else {
        callback();
    }
};
const checkCoreK = (rule, value, callback) => {
    if (value === 0) {
        callback()
    }
    if (!value) {
        callback(new Error('Please input the core level K'));
    }
    setTimeout(() => {
        if (!Number.isInteger(value)) {
            callback(new Error('Please input digits'))
        } else {
            if (value < 0) {
                callback(new Error('K must ≥ 0'))
            } else {
                callback()
            }
        }
    }, 1000)
};
const ruleForm = reactive({
    vertex: '',
    coreK: '',
    keywords: '',
});
const rules = reactive({
    vertex: [{validator: checkQueryVertex, trigger: 'blur'}],
    coreK: [{validator: checkCoreK, trigger: 'blur'}],
    keywords: [{validator: checkKeywords, trigger: 'blur'}],
});
const submitForm = (formEl) => {
    if (!formEl) return;
    formEl.validate((valid) => {
        if (valid) {
            console.log('submit!');
            axios.post('http://localhost:8080/graph/search', {
                vertex: ruleForm.vertex,
                coreK: ruleForm.coreK,
                keywords: ruleForm.keywords
            }).then(value => {
                const container = document.getElementById('container');
                container.innerHTML = "";
                if (document.getElementById('description') === null) {
                    const descriptionDiv = document.createElement('div');
                    descriptionDiv.id = 'description';
                    container.appendChild(descriptionDiv);
                }
                const descriptionDiv = document.getElementById('description');
                descriptionDiv.innerHTML = `正在渲染搜索数据，请稍后......`;

                let data = value.data.data
                console.log(data)

                const queryVertex = ruleForm.vertex;
                const communityKeywords = data.communityKeywords;
                const communityNum = communityKeywords.length;
                // ---筛选插件设置---
                const clusterMap = new Map();
                let clusterId = -1;
                const legendData = {
                    nodes: []
                }
                for (let i = 1; i <= communityNum; i++) {
                    legendData.nodes.push({"id": `${i}`, "label": `{${communityKeywords[i - 1]}}`});
                }
                legendData.nodes.forEach(node => {
                    node.size = 15;
                });
                // 动态生成filter函数
                const filterFC = {};
                for (let i = 1; i <= communityNum; i++) {
                    const cluster = `${i}`;
                    filterFC[cluster] = (d) => {
                        if (d.id === queryVertex) {
                            return true;
                        }
                        return d.cluster === cluster;
                    }
                }

                const legend = new G6.Legend({
                    data: legendData,
                    align: 'center',
                    layout: 'vertical', // vertical
                    position: 'bottom-left',
                    vertiSep: 6,
                    horiSep: 24,
                    offsetY: -24,
                    padding: [4, 16, 8, 16],
                    containerStyle: {
                        fill: '#ccc',
                        lineWidth: 1,
                    },
                    title: 'Community',
                    titleConfig: {
                        position: 'center',
                        offsetX: 0,
                        offsetY: 12,
                    },
                    filter: {
                        enable: true,
                        multiple: false,
                        trigger: 'click',
                        graphActiveState: 'activeByLegend',
                        graphInactiveState: 'inactiveByLegend',
                        filterFunctions: filterFC,
                    }
                });

                const graph = new G6.Graph({
                    container: 'container',
                    width: 1400,
                    height: 728,
                    fitCenter: true,
                    fitView: true,
                    fitViewPadding: [20, 40, 50, 20],
                    plugins: [tooltip, legend, minimap],
                    defaultNode: {
                        size: 5,
                        style: {
                            fill: '#C6E5FF',
                            stroke: '#5B8FF9',
                            lineWidth: 0.3,
                        },
                        labelCfg: {
                            style: {
                                fontSize: 3,
                                fill: '#000',
                            },
                            position: 'right',
                            offset: 1,
                        },
                    },
                    defaultEdge: {
                        size: 0.1,
                        color: '#333',
                    },
                    layout: {
                        type: 'gForce',
                        linkDistance: 100,
                        nodeStrength: 500,
                        preventOverlap: true,
                        nodeSpacing: 10,
                    },
                    modes: {
                        default: [
                            {
                                type: 'zoom-canvas',
                                enableOptimize: true,
                                optimizeZoom: 0.9,
                            },
                            {
                                type: 'drag-canvas',
                                enableOptimize: true,
                            },
                            'click-select',
                        ],
                    },
                    nodeStateStyles: {
                        activeByLegend: {
                            lineWidth: 2,
                            strokeOpacity: 0.5,
                            stroke: '#f00',
                            fill: '#FFBD32'
                        },
                        inactiveByLegend: {
                            opacity: 0.5,
                        },
                        selected: {
                            lineWidth: 2,
                        },
                        hover: {
                            fill: 'lightsteelblue',
                        },
                    },
                });

                data.graphData.nodes.forEach((node) => {
                    // 处理超长文本
                    node.label = fittingString(node.label, 100, globalFontSize);

                    // 获取筛选数据
                    if (node.id === queryVertex) {
                        if (!node.style) {
                            node.style = {};
                        }
                        node.style.fill = '#FFBD32'
                    }
                    if (node.cluster && clusterMap.get(node.cluster) === undefined) {
                        clusterMap.set(node.cluster, clusterId);
                        clusterId++;
                    }

                    // 获取节点的度
                    node.degree = 0;
                    data.graphData.edges.forEach((edge) => {
                        if (edge.source === node.id || edge.target === node.id) {
                            node.degree++;
                        }
                    });
                });
                let totalDegree = 0;
                data.graphData.nodes.forEach((node) => {
                    totalDegree += node.degree;
                })
                mapNodeSize(data.graphData.nodes, 'degree', [1, 10]);
                graph.clear()
                graph.data(data.graphData);
                graph.render();

                descriptionDiv.innerHTML = `搜索完毕！共搜索到${communityNum}个社区`;

                // 事件监听
                graph.on('node:mouseenter', (node) => {
                    const nodeItem = node.item;
                    graph.setItemState(nodeItem, 'hover', true);
                });
                graph.on('node:mouseleave', (node) => {
                    const nodeItem = node.item;
                    graph.setItemState(nodeItem, 'hover', false);
                });

                const clearStates = () => {
                    graph.getNodes().forEach((node) => {
                        graph.clearItemStates(node);
                    });
                    graph.getEdges().forEach((edge) => {
                        graph.clearItemStates(edge);
                    });
                };

                graph.on('canvas:click', () => {
                    clearStates();
                });

                function focusQueryNode() {
                    graph.focusItem(graph.findById(queryVertex), true, {
                        easing: 'easeCubic',
                        duration: 500,
                    });
                }

                // listen to the node click event
                graph.on('afterlayout', focusQueryNode);

                function handleNodeClick(event) {
                    const item = event.item;
                    graph.focusItem(item, true, {
                        easing: 'easeCubic',
                        duration: 500,
                    });
                }

                graph.on('node:click', handleNodeClick);

            }).catch(reason => {
                console.log(reason)
            })
        } else {
            console.log('error submit!');
            return false;
        }
    });
};
const resetForm = (formEl) => {
    if (!formEl) return;
    formEl.resetFields();
};
</script>

<style scoped>
.g6-component-tooltip {
    background-color: rgba(255, 255, 255, 0.8);
    padding: 0 10px 24px 10px;
    box-shadow: rgb(177, 176, 176) 0 0 10px;
}

.g6-minimap-viewport {
    border: 2px solid rgb(25, 128, 255);
}

.body {
    padding-left: 50px;
    position: relative;
    width: 1850px;
    height: 936px;
}

#container {
    float: left;
    height: 750px;
    width: 1400px;
    border: 2px solid;
}

#world-cloud {
    float: left;
    height: 750px;
    width: 400px;
    border: 2px solid;
    border-left: 0 solid;
}

.top {
    display: inline-block;
    vertical-align: middle;
}

.bottom {
    display: inline-block;
    vertical-align: middle;
    border: 2px solid;
    border-top: 0;
    width: 1802px;
    height: 180px;
}

.mapspace {
    float: left;
    height: 180px;
    width: 260px;
    border-right: 2px solid;
}

.input {
    position: relative;
    float: left;
    width: 1540px;
    height: 180px;
}

.title {
    position: absolute;
    right: 0;
    bottom: 0;
    font-size: 20px;
    font-family: "Microsoft YaHei UI", serif;
    font-weight: bold;
    color: white;
    background-color: black;
}

.upload-demo {
    position: absolute;
    width: 300px;
    left: 25%;
    top: 13%;
}

.demo-ruleForm {
    position: absolute;
    top: 1%;
    left: 50%;
    width: 400px;
    height: 180px;

}

.el-form-item {
    display: flex;
    --font-size: 14px;
    margin-bottom: 16px;
}
</style>