<template>
  <div>
    <slot></slot>
  </div>
</template>

<script>

  const extent =  [118.02252448821446,  27.04527653758214, 123.15774781361063, 31.18247145139634];
  const esrijsonFormat = new ol.format.EsriJSON();

  export default {
    name: 'Polygonlayer',
    props: ['param', 'index'],
    data() {
      return {
        _olLayer: null, // 用来在内部保存layer的引用
      }
    },
    created() {

      /**
       * 矢量图层
       */

       // 保存制图参数和图层对象的引用，加载完地图数据之后调用
      var param = this.param;
      var scope = this;

      /**
       * 检查字符串是否是数字
       * @param  {[type]} theObj [description]
       * @return {[type]}        [description]
       */
      function checkNumber(theObj) {
        var reg = /^[0-9]+.?[0-9]*$/;
        if (reg.test(theObj)) {
          return true;
        }
        return false;
      }


       var vectorSource = new ol.source.Vector({
       loader: function(extent, resolution, projection) {
         var url = param.url;
         $.ajax({url: url, dataType: 'jsonp', success: function(response) {
           if (response.error) {
             alert(response.error.message + '\n' +
                 response.error.details.join('\n'));
           } else {
             // dataProjection will be read from document
             var features = esrijsonFormat.readFeatures(response, {
               featureProjection: projection
             });
             if (features.length > 0) {
               vectorSource.addFeatures(features);
               scope.$emit( 'feaureLoaded', scope.index, features );
               // 保存可以用于分级设色的字段
               param.fields = [];
               for (var variable in features[0].values_) {
                 // 如果字段的属性值是数字的话，就可以用来做分级设色
                 if (features[0].values_.hasOwnProperty(variable) && checkNumber( features[0].values_[ variable ] ) ) {
                   let max, min;
                   max = min = features[0].values_[ variable ];
                   features.forEach( function( feature, index ) {
                     if( +feature.values_[ variable ] > max ) {
                       max = +feature.values_[ variable ];
                     }
                     if( +feature.values_[ variable ] < min ) {
                       min = +feature.values_[ variable ];
                     }
                   } )

                   // 保存可以用来做分级设色的字段（值为数字）和该字段值的最大值和最小值
                   param.fields.push( {
                     field: variable,
                     max: max,
                     min: min,
                   } );
                 }
               }
             }
             // 通知顶层数据已经添加到地图上了
             param.container.doneRequestParams++;
           }
         }});
       },
     });

     var scope = this;

     // 初始的样式
     var initialStyle = new ol.style.Style({
       fill: new ol.style.Fill( this.param )
     });

     this._olLayer = new ol.layer.Vector({
       source: vectorSource,
       style: initialStyle
     });

      // 图层对象都要定义一个update函数，用于实时更新制图参数
      this._olLayer.update = function() {

        /**
         * 根据分段数、分级方法和分级字段生成分段区间
         * @return {[type]} [description]
         */
        function getNodeArray() {
          // 分级数
          var count = param.gradeCount;
          // 获取当前分级字段，以及其最大值最小值
          var field = param.fields.find( function( element, index ){
            return element.field === param.gradeField;
          } )
          /*
          field {
            field: variable,
            max: max,
            min: min,
          }
           */
          // 当前字段的最大值和最小值
          var max = field.max;
          var min = field.min;
          // 保存分级节点的数组，分级间距
          // 节点数组包含首尾
          var a = [], gap = 0;
          switch ( param.gradeMethod ) {
            case '等间距分段':
              gap = ( max - min ) / count;
              while ( min < max ) {
                a.push( min );
                min += gap;
              }
              a.push( max );
              return a;
            default:

          }
        }

        /**
         * 将#ccc补全为#cccccc
         * @param  {[type]} color [description]
         * @return {[type]}       [description]
         */
        function fix16Color( color ){
          if( color.length < 7 ) {
            return '#' + color.slice(1).split( '' ).map( function( value ) { return value + value } ).join( '' );
          } else {
            return color;
          }
        }

        /**
         * 获取颜色数据   16进制 #ffffff
         * @returns {Array}
         */
        function getColorArray16(){
          var customNum = param.gradeCount;
          var colorLow = fix16Color( param.startColor );
          var colorHigh = fix16Color( param.stopColor );
          var rl = parseInt(colorLow.substr(1,2),16);
          var gl = parseInt(colorLow.substr(3,2),16);
          var bl = parseInt(colorLow.substr(5,2),16);
          var rh = parseInt(colorHigh.substr(1,2),16);
          var gh = parseInt(colorHigh.substr(3,2),16);
          var bh = parseInt(colorHigh.substr(5,2),16);
          var colors = [];
          if(customNum==1){
              colors.push(colorLow);
          }else if(customNum>1){
              for(var i=0;i<customNum;i++){
                  var r = parseInt(rl + (rh-rl)*i/(customNum<2?1:(customNum-1)));
                  var g = parseInt(gl + (gh-gl)*i/(customNum<2?1:(customNum-1)));
                  var b = parseInt(bl + (bh-bl)*i/(customNum<2?1:(customNum-1)));
                  colors.push("#"+(r.toString(16).length!=2?"0"+r.toString(16):r.toString(16))
                      +""+(g.toString(16).length!=2?"0"+g.toString(16):g.toString(16))
                      +""+(b.toString(16).length!=2?"0"+b.toString(16):b.toString(16)))
              }
          }
          return colors;
        }

        /**
         * 获取分级设色后的样式数组
         * @return {[type]} [description]
         */
        function styleGenerator() {
          // 如果是分级设色的话
          if( param.isGrade ) {
            let styles = [];
            // 获取颜色数组
            let colors = getColorArray16();
            // 获取分级节点数组
            let nodes = getNodeArray();

            /*
            分级的属性值处于nodes[i]和nodes[i+1]之间（左开右闭）的面状要素
            将会被渲染成colors[i]
             */

            for( let i = 0; i < param.gradeCount; i++ ) {
              styles.push(
                new ol.style.Style({
                  fill: new ol.style.Fill( { color: colors[ i ] } ),
                  geometry: function(feature) {
                    // 每次重绘都会调用（拖拽、缩放都包括）
                    // feature代表单个矢量要素
                    // feature.values_能读取到要素的字段信息

                    if( i === 0 && feature.values_[ param.gradeField ] === nodes[0] )
                      return feature.getGeometry();

                    if( nodes[ i ] < feature.values_[ param.gradeField ] && nodes[ i + 1 ] >= feature.values_[ param.gradeField ] )
                      return feature.getGeometry();
                  }
                })
              )
            }

            return styles;
          }
          // 如果只是普通的面状图层
          else {
            return new ol.style.Style({
              fill: new ol.style.Fill( param )
            });
          }
        }

        return function() {
          // 更新图层样式（包括分级设色和普通面状图层的样式）
          this.setStyle( styleGenerator() );
          // 图层是否可见
          param.visible? this.setVisible( true ) : this.setVisible( false );
        }
      }()

      // 将制图参数和图层对象绑定
      this.param.__myob__.dep.addSub( this._olLayer );

      // 将当前图层添加到ol.Map中
      this.$nextTick(t => this.$parent.$emit("addtile", this._olLayer));

    },
  }
</script>
