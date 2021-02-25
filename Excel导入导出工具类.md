# Excel导入导出工具类




```javascript
// https://github.com/SheetJS/sheetjs
import XLSX from 'xlsx';

export default class ExcelUtil extends Object {
  /**
   * export file
   * @param header [{ key: 'name', title: '姓名' }, { key: 'sex', title: '性别' }]
   * @param data [{ name: '张三', sex: '男', age: '20' }, { name: '李四', sex: '女', age: '18' }]
   * @param fileName
   */
  static export(header, data, fileName = '记录表.xlsx') {
    let header2 = header
      .map(
        (item, i) =>
          Object.assign({}, { [String.fromCharCode(65 + i) + 1]: { k: item.key, v: item.title } })
        // { A1: { k: 'name', v: '姓名' } }
      )
      // [{ A1: { k: 'name', v: '姓名' } }, { B1: { k: 'sex', v: '性别' } }]
      .reduce((prev, next) => Object.assign({}, prev, next));
    // {
    //   A1: { k: 'name', v: '姓名' }, B1: { k: 'sex', v: '性别' }
    // }

    let rows = {};
    if (data && data.length > 0) {
      rows = data
        .map(
          (item, i) =>
            header.map(
              (elem, j) =>
                Object.assign(
                  {},
                  { [String.fromCharCode(65 + j) + (i + 2)]: { v: item[elem.key] } }
                )
              // { A2: { v: '张三' } }
            )
          // [{ A2: { v: '张三' } }, { B2: { v: '男' } }]
        )
        // [
        //   [{ A2: { v: '张三' } }, { B2: { v: '男' } }],
        //   [{ A3: { v: '李四' } }, { B3: { v: '女' } }]
        // ]
        .reduce((prev, next) => prev.concat(next))
        // [
        //   { A2: { v: '张三' } }, { B2: { v: '男' } },
        //   { A3: { v: '李四' } }, { B3: { v: '女' } }
        // ]
        .reduce((prev, next) => Object.assign({}, prev, next));
      // {
      //   A2: { v: '张三' }, B2: { v: '男' },
      //   A3: { v: '李四' }, B3: { v: '女' }
      // }
    }

    let lines = Object.assign({}, header2, rows);
    // {
    //   A1: { k: 'name', v: '姓名' }, B1: { k: 'sex', v: '性别' },
    //   A2: { v: '张三' }, B2: { v: '男' },
    //   A3: { v: '李四' }, B3: { v: '女' }
    // }
    let position = Object.keys(lines);
    // [A1, B1, A2, B2, A3, B3];
    let ref = `${position[0]}:${position[position.length - 1]}`;
    // 'A1:B3'

    let cols = header.map((item, i) => Object.assign({}, { wpx: 150 }));

    let workbook = {
      SheetNames: ['Sheet'],
      Sheets: {
        Sheet: Object.assign({}, lines, {
          '!ref': ref,
          '!cols': cols,
        }),
      },
    };

    XLSX.writeFile(workbook, fileName);
  }

  /**
   * import file
   * <input type='file' accept='.xlsx, .xls' onChange={(e)=>{ExcelUtil.import(e)} }/>
   * @param file
   */
  static import(file) {
    let { files } = file.target;
    let reader = new FileReader();
    reader.onload = event => {
      try {
        let { result } = event.target;
        let workbook = XLSX.read(result, { type: 'binary' });
        let data = [];
        for (let sheet in workbook.Sheets) {
          if (workbook.Sheets.hasOwnProperty(sheet)) {
            data = data.concat(XLSX.utils.sheet_to_json(workbook.Sheets[sheet]));
          }
        }
        console.log(data);
      } catch (e) {
        console.log('文件类型不正确');
        return;
      }
    };
    reader.readAsBinaryString(files[0]);
  }
}
```



## 参考

【1】[《React导入导出Excle》](https://www.cnblogs.com/yuyuan-bb/p/10965104.html) 