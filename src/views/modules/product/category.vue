<template>
    <div>

        <el-switch
            v-model="draggable"
            active-text="开启拖拽"
            inactive-text="关闭拖拽"
        >
        </el-switch>

        <el-button @click="batchSave" v-if="draggable">批量保存</el-button>
        <el-button type="danger" @click="batchDelete">批量删除</el-button>

        <el-tree
            :data="menus"
            :props="defaultProps"
            :expand-on-click-node="false"
            show-checkbox
            node-key="catId"
            :default-expanded-keys="expandedKey"
            :draggable="draggable"
            :allow-drog="allowDrop"
            @node-drop="handleDrop"
            ref="menuTree"
        >
            <span
                class="custom-tree-node"
                slot-scope="{ node, data }"
            >
                <span>{{ node.label }}</span>

                <span>
                    <el-button
                        type="text"
                        size="mini"
                        @click="() => append(data)"
                        v-if="node.level <= 2"
                    >
                        Append
                    </el-button>

                    <el-button
                        type="text"
                        size="mini"
                        @click="edit(data)"
                    >
                        Edit
                    </el-button>

                    <el-button
                        type="text"
                        size="mini"
                        @click="() => remove(node, data)"
                        v-if="node.childNodes.length==0"
                    >
                        Delete
                    </el-button>
                </span>
            </span>
        </el-tree>

        <el-dialog
            :title="title"
            :visible.sync="dialogVisible"
            width="30%"
            :close-on-click-modal="false"
        >
            <el-form :model="category">
                <el-form-item label="分类名称">
                    <el-input
                        v-model="category.name"
                        autocomplete="off"
                    ></el-input>
                </el-form-item>

                <el-form-item label="图标">
                    <el-input
                        v-model="category.icon"
                        autocomplete="off"
                    ></el-input>
                </el-form-item>

                <el-form-item label="计量单位">
                    <el-input
                        v-model="category.productUnit"
                        autocomplete="off"
                    ></el-input>
                </el-form-item>
            </el-form>
            <span
                slot="footer"
                class="dialog-footer"
            >
                <el-button @click="dialogVisible = false">取 消</el-button>
                <el-button
                    type="primary"
                    @click="submitData"
                >确 定</el-button>
            </span>
        </el-dialog>
    </div>
</template>

<script>
export default {
    data() {
        return {
            category: {
                name: "",
                parentCid: 0,
                catLevel: 0,
                showStatus: 1,
                sort: 0,
                catId: null,
                icon: "",
                productUnit: "",
            },
            menus: [],
            expandedKey: [],
            defaultProps: {
                children: "children",
                label: "name",
            },
            dialogVisible: false,
            dialogType: "",
            title: "",
            updateNodes: [],
            draggable: false,
            pCid: [],
            maxLevel: 0,
        };
    },
    methods: {
        getMenus() {
            this.$http({
                url: this.$http.adornUrl("/product/category/list/tree"),
                method: "get",
            }).then(({ data }) => {
                console.log("成功获取到菜单数据...", data.data);
                this.menus = data.data;
            });
        },

        append(data) {
            console.log("appen", data);
            this.dialogVisible = true;
            this.category.parentCid = data.catId;
            this.category.catLevel = data.catLevel * 1 + 1;
            this.dialogType = "add";
            this.title = "添加分类";

            this.category.name = null;
            this.category.catId = null;
            this.category.icon = "";
            this.category.productUnit = "";
            this.category.sort = 0;
            this.category.showStatus = 1;
        },

        remove(node, data) {
            var ids = [data.catId];
            this.$confirm(`是否删除[${data.name}]菜单?`, "提示", {
                confirmButtonText: "确定",
                cancelButtonText: "取消",
                type: "warning",
            })
                .then(() => {
                    this.$http({
                        url: this.$http.adornUrl("/product/category/delete"),
                        method: "post",
                        data: this.$http.adornData(ids, false),
                    }).then(({ data }) => {
                        this.$message({
                            message: "菜单删除成功",
                            type: "success",
                        });
                        this.getMenus();
                        this.expandedKey = [node.parent.data.catId];
                    });
                })
                .catch(() => {});
            console.log("remove", node, data);
        },

        addCategory() {
            this.$http({
                url: this.$http.adornUrl("/product/category/save"),
                method: "post",
                data: this.$http.adornData(this.category, false),
            }).then(({ data }) => {
                this.$message({
                    message: "菜单保存成功",
                    type: "success",
                });
                this.dialogVisible = false;
                this.getMenus();
                this.expandedKey = [this.category.parentCid];
            });
        },

        edit(data) {
            this.dialogType = "edit";
            this.title = "修改分类";
            this.dialogVisible = true;
            //发送请求获取当前节点最新的数据
            this.$http({
                url: this.$http.adornUrl(
                    `/product/category/info/${data.catId}`
                ),
                method: "get",
            }).then(({ data }) => {
                console.log(data);
                this.category.name = data.category.name;
                this.category.catId = data.category.catId;
                this.category.icon = data.category.icon;
                this.category.productUnit = data.category.productUnit;
                this.category.parentCid = data.category.parentCid;
                this.category.catLevel = data.category.catLevel;
                this.category.sort = data.category.sort;
                this.category.showStatus = data.category.showStatus;
            });
        },

        editCategory() {
            var { catId, name, icon, productUnit } = this.category;
            this.$http({
                url: this.$http.adornUrl("/product/category/update"),
                method: "post",
                data: this.$http.adornData(
                    { catId, name, icon, productUnit },
                    false
                ),
            }).then(({ data }) => {
                this.$message({
                    message: "菜单修改成功",
                    type: "success",
                });
                this.dialogVisible = false;
                this.getMenus();
                this.expandedKey = [this.category.parentCid];
            });
        },

        submitData() {
            if (this.dialogType == "add") {
                this.addCategory();
            }

            if (this.dialogType == "edit") {
                this.editCategory();
            }
        },

        allowDrop(draggingNode, dropNode, type) {
            console.log(draggingNode);
            console.log(dropNode);
            console.log(type);
            //1.被拖动的当前节点以及所在父亲点总层数不能大于3
            this.countNodeLevel(draggingNode);
            //当前拖动的节点加父节点不大于3
            let deep = Math.abs(this.maxLevel - draggingNode.level) + 1;
            if (type == "inner") {
                return deep + dropNode.level <= 3;
            } else {
                return deep + dropNode.parent.level <= 3;
            }
        },

        countNodeLevel(node) {
            console.log(node);
            this.maxLevel = node.catLevel;
            if (node.childrenNodes != null && node.children.length > 0) {
                for (let i = 0; i < node.children.length; ++i) {
                    if (node.children[i].level > this.maxLevel) {
                        this.maxLevel = node.children[i].level;
                    }
                    this.countNodeLevel(node.children[i]);
                }
            }
        },

        handleDrop(draggingNode, dropNode, dropType, type) {
            //1.当前节点最新的父节点ID
            let pCid = 0;
            let siblings = null;
            if (dropType === "before" || dropType === "after") {
                pCid =
                    dropNode.parent.data.catId == undefined
                        ? 0
                        : dropNode.parent.data.catId;
                siblings = dropNode.parent.childNodes;
            } else {
                pCid = dropNode.data.catId;
                siblings = dropNode.parent.childNodes;
            }

            this.pCid.push(pCid);

            //2.当前节点最新顺序

            for (let i = 0; i < siblings.length; ++i) {
                if (siblings[i].data.catId === draggingNode.data.catId) {
                    //如果遍历的是拖拽的节点
                    let catLevel = draggingNode.level;
                    if (siblings[i].level != draggingNode.level) {
                        //当前节点的层级发生变化
                        catLevel = siblings[i].level;
                        //修改子节点层级
                        this.updateChildNodeLevel(siblings[i]);
                    }
                    this.updateNodes.push({
                        catId: siblings[i].data.catId,
                        sort: i,
                        parentCid: pCid,
                        catLevel,
                    });
                } else {
                    this.updateNodes.push({
                        catId: siblings[i].data.catId,
                        sort: i,
                    });
                }
            }
            //3.当前节点最新层级
            
        },

        updateChildNodeLevel(node) {
            if (node.childNodes.length > 0) {
                for (let i = 0; i < node.childNodes.length; ++i) {
                    var cNode = node.childNodes[i].data;
                    this.updateNodes.push({
                        catId: cNode.catId,
                        catLevel: node.childNodes[i].level,
                    });
                    this.updateChildNodeLevel(node.childNodes[i]);
                }
            }
        },

        batchSave() {
            this.$http({
                url: this.$http.adornUrl("/product/category/update/sort"),
                method: "post",
                data: this.$http.adornData(this.updateNodes, false),
            }).then(({ data }) => {
                this.$message({
                    message: "顺序修改成功",
                    type: "success",
                });
                this.getMenus();
                this.maxLevel = 0;
                this.updateNodes = [];
                this.expandedKey = this.pCid;
                //this.pCid = 0;
            });
        },

        batchDelete() {
            let catIds = [];
            let nodes = this.$refs.menuTree.getCheckedNodes();
            for (let i = 0; i < nodes.length; ++i) {
                catIds.push(nodes[i].catId);
            }

            this.$confirm(`是否批量删除[${catIds}]菜单?`, "提示", {
                confirmButtonText: "确定",
                cancelButtonText: "取消",
                type: "warning",
            }).then(() => {
                    this.$http({
                        url: this.$http.adornUrl("/product/category/delete"),
                        method: "post",
                        data: this.$http.adornData(catIds, false),
                    }).then(({ data }) => {
                        this.$message({
                            message: "菜单批量删除成功",
                            type: "success",
                        });
                        this.getMenus();
                    });
            }).catch(() => {});

        },

    },

    created() {
        this.getMenus();
    },

    mounted() {},
};
</script>

<style lang="scss" scoped>
</style>