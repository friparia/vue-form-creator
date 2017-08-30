<template>
    <div @mouseleave="cancelDrag" @mousemove="drag" @mouseup="stopDrag" style="height:100%" class="unselect">
        <el-row :gutter="20">
            <el-col :span="18">
                <el-card class="box-card" style="padding-top:50px;">
                    <el-form label-width="120px" id="custom-form">
                        <template
                            v-for="field in form.fields"
                        >
                            <div @click="selectField(field)" v-bind:class="formActive(field)" @mousedown="startFieldDrag(field, $event)" class="field" :data-position="field.position">
                                <div class="divider" v-show="dragBox.dropPosition == field.position"></div>
                                <div v-if="field.type == 'legend'" class="wrd-title" style="display:inline-block;">{{ getName(field) }}</div>
                                <el-form-item v-if="field.type != 'legend'" class="form-element" :label="getName(field)" style="margin-bottom:5px;margin-top:5px;display:inline-block;">
                                    <el-input
                                        v-if="field.type == 'text'"
                                        :placeholder="getDescription(field)"
                                    >
                                    </el-input>
                                </el-form-item>
                                <el-button size="common" class="delete" type="text" @click="deleteField(field)"><i class="el-icon-close"></i></el-button>
                            </div>
                        </template>
                        <el-form-item class="field" data-position="0">
                            <div class="divider" v-show="dragBox.dropPosition == 0"></div>
                            <el-button type="primary">保存</el-button>
                        </el-form-item>
                    </el-form>
                </el-card>
            </el-col>
            <el-col :span="6">
                <el-card class="box-card">
                    <div v-for="field in fields" class="text item">
                        <el-col :span="12" style="padding:0;">
                            <div class="field-element" @mousedown="startNewFieldDrag(field, $event)">
                                {{ field.name }}
                            </div>
                        </el-col>
                    </div>
                </el-card>
                <el-card class="box-card" style="margin-top:100px;">
                    <div
                        v-if="selectedField.type == 'legend'"
                    >
                        <h4>基本信息</h4>
                        <el-form
                            label-width="80px"
                            action="#"
                        >
                            <el-form-item
                                label="字段名"
                            >
                                <el-input v-model="selectedField.properties.name.value" @blur="updateProperty(selectedField.properties.name)"></el-input>
                            </el-form-item>
                        </el-form>
                    </div>
                    <div
                        v-if="selectedField.type == 'text'"
                    >
                        <h4>基本信息</h4>
                        <el-form
                            action="#"
                            label-width="80px"
                        >
                            <el-form-item
                                label="字段名"
                            >
                                <el-input v-model="selectedField.properties.name.value" @blur="updateProperty(selectedField.properties.name)"></el-input>
                            </el-form-item>
                        </el-form>
                    </div>
                </el-card>
            </el-col>
        </el-row>
        <div id="drag-box" v-show="dragging" :style="{left: dragBox.x + 'px', top: dragBox.y + 'px'}" v-html="dragBox.content">
        </div>
    </div>
</template>

<script>
 export default{
     data() {
         return {
             form: {},
             fields: {},
             dragging: false,
             timer: 0,
             dragBox: {
                 x: 0,
                 y: 0,
                 content: '',
                 id: 0,
                 canDrop: false,
                 dropPosition: -1,
                 type: 'text',
             },
             selectedField: {},
             isNew: false,
         }
     },
     created() {
         this.fetchData();
         this.fetchFields();
     },
     methods: {
         formActive: function(field){
             return {
                 active: this.selectedField.id == field.id,
             };
         },
         fetchData(){
             var id = this.$route.params.id;
             this.$http.get('/api/forms/' + id).then((res) => {
                 this.form = res.body;
             });
         },
         fetchFields(){
             this.$http.get('/api/fields').then((res) => {
                 this.fields = res.body;
             });
         },
         startFieldDrag(field, e){
             this.timer = setTimeout(() => {
                 this.isNew = false;
                 this.dragging = true;
                 this.dragBox.content = this.getName(field);
                 this.dragBox.x = e.clientX + 5;
                 this.dragBox.y = e.clientY - 130;
                 this.dragBox.type = field.type;
                 this.dragBox.id = field.id;
             }, 200);
         },
         startNewFieldDrag(field, e){
             this.timer = setTimeout(() => {
                 this.isNew = true;
                 this.dragging = true;
                 this.dragBox.content = field.name;
                 this.dragBox.x = e.clientX + 5;
                 this.dragBox.y = e.clientY - 35;
                 this.dragBox.type = field.type;
             }, 200);
         },
         drag(e){
             if(this.dragging){
                 this.dragBox.dropPosition = -1;
                 this.dragBox.x = e.clientX + 5;
                 this.dragBox.y = e.clientY - 35;
                 var that = this;
                 $('.field').each(function(i, el){
                     var elementTop = $(el).position().top + 30;
                     var elementBottom = $(el).position().top + 30 + $(el).height();
                     if(e.clientY > elementTop && e.clientY < elementBottom){
                         that.dragBox.canDrop = true;
                         that.dragBox.dropPosition = $(el).data('position');
                     }
                 });
             }
         },
         stopDrag(){
             if(this.dragging){
                 this.dragging = false;
                 if(this.dragBox.canDrop){
                     this.dragBox.canDrop = false;
                     var position = this.dragBox.dropPosition;
                     if(position == 0){
                         var count = 1;
                         for(var i in this.form.fields){
                             count++;
                         }
                         this.dragBox.dropPosition = count;
                     }
                     if(position == -1){
                         console.log('error');
                         return;
                     }
                     if(this.isNew){
                         this.$http.post('/api/fields', {
                             form_id: this.form.id,
                             type: this.dragBox.type,
                             position: this.dragBox.dropPosition,
                         }).then((res) => {
                             this.fetchData();
                         });
                     }else{
                         this.$http.put('/api/fields/' + this.dragBox.id, {
                             type: this.dragBox.type,
                             position: this.dragBox.dropPosition,
                         }).then((res) => {
                             this.fetchData();
                         });
                     }
                 }
             }
             this.cancelDrag();
         },
         cancelDrag(){
             this.dragging = false;
             this.dragBox.canDrop = false;
             this.dragBox.dropPosition = -1;
             clearTimeout(this.timer);
         },
         getDescription(field){
             for(var i in field.properties){
                 var property = field.properties[i];
                 if(property.name == 'description'){
                     return property.value;
                 }
             }
             return "";
         },
         getName(field){
             for(var i in field.properties){
                 var property = field.properties[i];
                 if(property.name == 'name'){
                     return property.value;
                 }
             }
             return "";
         },
         selectField(field){
             this.selectedField = Object.assign({}, field);
             var properties = {}
             for(var i in field.properties){
                 var property = field.properties[i];
                 properties[property.name] = {value: property.value, id: property.id}
             }
             this.selectedField.properties = properties;
         },
         deleteField(field){
             this.$http.delete('/api/fields/' + field.id).then((res) => {
                 this.fetchData();
             });
         },
         updateProperty(property){
             this.$http.put('/api/properties/' + property.id, {value: property.value}).then((res) => {
                 this.fetchData();
             });
         }
     }
 }
</script>
<style>
 .field-element{
     padding: 10px;
     cursor: pointer;
 }
 .unselect{
     -webkit-user-select: none;  /* Chrome all / Safari all */
     -moz-user-select: none;     /* Firefox all */
     -ms-user-select: none;      /* IE 10+ */
     user-select: none;          /* Likely future */
 }
 .divider{
     border:1px solid;
     height:0px;
     margin-bottom:3px;
 }
 .form-element{
     width: 500px;
     margin: auto;
     -webkit-user-select: none;  /* Chrome all / Safari all */
     -moz-user-select: none;     /* Firefox all */
     -ms-user-select: none;      /* IE 10+ */
     user-select: none;          /* Likely future */
 }
 #drag-box{
     position:absolute;
     border: dotted 1px;
 }
 .field:hover{
     background: rgb(229, 243, 255);
 }
 .field.active:hover{
     background: rgb(204, 232, 255);
 }
 .active{
     background: rgb(204, 232, 255);
     border: 1px solid rgb(153, 209, 255);
 }
</style>
