
书接上回，我们今天继续讲解实现对象集合与DataTable的相互转换。


![](https://img2024.cnblogs.com/blog/386841/202412/386841-20241201214519100-1188328064.png)


# ***01***、把表格转换为对象集合


该方法是将表格的列名称作为类的属性名，将表格的行数据转为类的对象。从而实现表格转换为对象集合。同时我们约定如果类的属性设置了DescriptionAttribute特性，则特性值和表格列名一一对应，如果没有设置特性则取属性名称和列名一一对应。


同时我们需要约束类只能为结构体或类，而不能是枚举、基础类型、以及集合类型、委托、接口等。


类的类型校验成功后，我们还需要校验表格是否能转换为对象，即判断表格列名和类的属性名称或者Description特性值是否存在一致，如果没有一个表格列名和类的属性能对应上，则报表格列名无法映射至对象属性，无法完成转换异常。


当这些校验成功后，开始循环处理表格行记录，把每一行都转换为一个对象。


我们可以通过反射动态实例化对象，再通过反射对对象的属性动态赋值。因为我们的对象即支持类也支持结构体，因此这里面就会遇到一个技术问题，正常的property.SetValue方法并没有办法给结构体动态赋值。


这是因为结构体是值类型，而property.SetValue方法的参数都是object，因此这里面就涉及到装箱拆箱，因此SetValue是设置了装箱以后的对象，而并不能改变原对象。


而解决办法就是先把结构体赋值给object变量，然后对object变量进行SetValue设置值，最后再把object变量转为结构体。


下面我们一起看看具体实现代码：



```
//把表格转换为对象集合
//如果设置DescriptionAttribute，则将特性值作为表格的列名称
//否则将属性名作为表格的列名称
public static IEnumerable<T> ToModels<T>(DataTable dataTable)
{
    //T必须是结构体或类，并且不能是集合类型
    AssertTypeValid();
    if (0 == dataTable.Rows.Count)
    {
        return [];
    }
    //获取T所有可写入属性
    var properties = typeof(T).GetProperties().Where(u => u.CanWrite);
    //校验表格是否能转换为对象
    var isCanParse = IsCanMapDataTableToModel(dataTable, properties);
    if (!isCanParse)
    {
        throw new NotSupportedException("The column name of the table cannot be mapped to an object property, and the conversion cannot be completed.");
    }
    var models = new List();
    foreach (DataRow dr in dataTable.Rows)
    {
        //通过反射实例化T
        var model = Activator.CreateInstance();
        //把行数据映射到对象上
        if (typeof(T).IsClass)
        {
            //处理T为类的情况
            MapRowToModel(dr, model, properties);
        }
        else
        {
            //处理T为结构体的情况
            object boxed = model!;
            MapRowToModel<object>(dr, boxed, properties);
            model = (T)boxed;
        }
        models.Add(model);
    }
    return models;
}
//校验表格是否能转换为对象
private static bool IsCanMapDataTableToModel(DataTable dataTable, IEnumerable properties)
{
    var isCanParse = false;
    foreach (var property in properties)
    {
        //根据属性获取列名
        var columnName = GetColumnName(property);
        if (!dataTable.Columns.Contains(columnName))
        {
            continue;
        }
        isCanParse = true;
    }
    return isCanParse;
}
//把行数据映射到对象上
private static void MapRowToModel<T>(DataRow dataRow, T model, IEnumerable properties)
{
    foreach (var property in properties)
    {
        //根据属性获取列名
        var columnName = GetColumnName(property);
        if (!dataRow.Table.Columns.Contains(columnName))
        {
            continue;
        }
        //获取单元格值
        var value = dataRow[columnName];
        if (value != DBNull.Value)
        {
            //给对象属性赋值
            property.SetValue(model, Convert.ChangeType(value, property.PropertyType));
        }
    }
}

```

我们做个简单的单元测试：



```
[Fact]
public void ToModels()
{
    //验证正常情况
    var table = TableHelper.Createdouble>>();
    var row1 = table.NewRow();
    row1[0] = "Id-11";
    row1[1] = "名称-12";
    row1[2] = 33.13;
    table.Rows.Add(row1);
    var row2 = table.NewRow();
    row2[0] = "Id-21";
    row2[1] = "名称-22";
    row2[2] = 33.23;
    table.Rows.Add(row2);
    var students = TableHelper.ToModelsdouble>>(table);
    Assert.Equal(2, students.Count());
    Assert.Equal("Id-11", students.ElementAt(0).Id);
    Assert.Equal("名称-12", students.ElementAt(0).Name);
    Assert.Equal(33.13, students.ElementAt(0).Age);
    Assert.Equal("Id-21", students.ElementAt(1).Id);
    Assert.Equal("名称-22", students.ElementAt(1).Name);
    Assert.Equal(33.23, students.ElementAt(1).Age);
}

```

# ***02***、把对象集合转换为表格


该方法首先会调用根据对象创建表格方法得到一个空白表格，然后通过反射获取对象的所有属性，然后循环处理对象集合，把一个对象的所有属性值一个一个添加行的所有列中，这样就完成了一个对象映射成表的一行记录，直至所有对象转换完成即可得到一个表格。


代码如下：



```
//把对象集合转为表格
//如果设置DescriptionAttribute，则将特性值作为表格的列名称
//否则将属性名作为表格的列名称
public static DataTable ToDataTable<T>(IEnumerable models, string? tableName = null)
{
    //创建表格
    var dataTable = Create(tableName);
    if (models == null || !models.Any())
    {
        return dataTable;
    }
    //获取所有属性
    var properties = typeof(T).GetProperties().Where(u => u.CanRead);
    foreach (var model in models)
    {
        //创建行
        var dataRow = dataTable.NewRow();
        foreach (var property in properties)
        {
            //根据属性获取列名
            var columnName = GetColumnName(property);
            //填充行数据
            dataRow[columnName] = property.GetValue(model);
        }
        dataTable.Rows.Add(dataRow);
    }
    return dataTable;
}

```

进行如下单元测试：



```
[Fact]
public void ToDataTable()
{
    //验证正常情况
    var students = new Listdouble>>();
    var student1 = new Student<double>
    {
        Id = "Id-11",
        Name = "名称-12",
        Age = 33.13
    };
    students.Add(student1);
    var student2 = new Student<double>
    {
        Id = "Id-21",
        Name = "名称-22",
        Age = 33.23
    };
    students.Add(student2);
    var table = TableHelper.ToDataTabledouble>>(students, "学生表");
    Assert.Equal("学生表", table.TableName);
    Assert.Equal(2, table.Rows.Count);
    Assert.Equal("Id-11", table.Rows[0][0]);
    Assert.Equal("名称-12", table.Rows[0][1]);
    Assert.Equal("33.13", table.Rows[0][2].ToString());
    Assert.Equal("Id-21", table.Rows[1][0]);
    Assert.Equal("名称-22", table.Rows[1][1]);
    Assert.Equal("33.23", table.Rows[1][2].ToString());
}

```

# ***03***、把一维数组作为一列转换为表格


该方法比较简单就是把一个一维数组作为一列数据创建一张表格，同时可以选择是否填写表名和列名。具体代码如下：



```
//把一维数组作为一列转换为表格
public static DataTable ToDataTableWithColumnArray<TColumn>(TColumn[] array, string? tableName = null, string? columnName = null)
{
    var dataTable = new DataTable(tableName);
    //创建列
    dataTable.Columns.Add(columnName, typeof(TColumn));
    //添加行数据
    foreach (var item in array)
    {
        var dataRow = dataTable.NewRow();
        dataRow[0] = item;
        dataTable.Rows.Add(dataRow);
    }
    return dataTable;
}

```

单元测试如下：



```
[Fact]
public void ToDataTableWithColumnArray()
{
    //验证正常情况
    var columns = new string[] { "A", "B" };
    var table = TableHelper.ToDataTableWithColumnArray<string>(columns, "学生表");
    Assert.Equal("学生表", table.TableName);
    Assert.Equal("Column1", table.Columns[0].ColumnName);
    Assert.Equal(2, table.Rows.Count);
    Assert.Equal("A", table.Rows[0][0]);
    Assert.Equal("B", table.Rows[1][0]);
    table = TableHelper.ToDataTableWithColumnArray<string>(columns, "学生表", "列");
    Assert.Equal("列", table.Columns[0].ColumnName);
}

```

# ***04***、把一维数组作为一行转换为表格


该方法也比较简单就是把一个一维数组作为一行数据创建一张表格，同时可以选择是否填写表名。具体代码如下：



```
//把一维数组作为一行转换为表格
public static DataTable ToDataTableWithRowArray<TRow>(TRow[] array, string? tableName = null)
{
    var dataTable = new DataTable(tableName);
    //创建列
    for (var i = 0; i < array.Length; i++)
    {
        dataTable.Columns.Add(null, typeof(TRow));
    }
    //添加行数据
    var dataRow = dataTable.NewRow();
    for (var i = 0; i < array.Length; i++)
    {
        dataRow[i] = array[i];
    }
    dataTable.Rows.Add(dataRow);
    return dataTable;
}

```

# ***05***、行列转置


该方法是指把DataTable中的行和列互换，就是行的数据变成列，列的数据变成行。如下图示例：


![](https://img2024.cnblogs.com/blog/386841/202412/386841-20241201214509388-1723508055.png)


这个示例转换，第一个表格中列名并没有作为数据进行转换，因此我们会提供一个可选项参数用来指示要不要把类目作为数据进行转换。


整个方法实现逻辑也很简单，就是以原表格行数为列数创建一个新表格，然后在循环处理原表格列，并把原表格一列数据填充至新表格的一行数据中，直至原表格所有列处理完成则完成行列转置。具体代码如下：



```
//行列转置
public static DataTable Transpose(DataTable dataTable, bool isColumnNameAsData = true)
{
    var transposed = new DataTable(dataTable.TableName);
    //如果列名作为数据，则需要多加一列
    if (isColumnNameAsData)
    {
        transposed.Columns.Add();
    }
    //转置后，行数即为新的列数
    for (int i = 0; i < dataTable.Rows.Count; i++)
    {
        transposed.Columns.Add();
    }
    //以列为单位，一次处理一列数据
    for (var column = 0; column < dataTable.Columns.Count; column++)
    {
        //创建新行
        var newRow = transposed.NewRow();
        //如果列名作为数据，则先把列名加入第一列
        if (isColumnNameAsData)
        {
            newRow[0] = dataTable.Columns[column].ColumnName;
        }
        //把一列数据转为一行数据
        for (var row = 0; row < dataTable.Rows.Count; row++)
        {
            //如果列名作为数据，则行数据从第二列开始填充
            var rowIndex = isColumnNameAsData ? row + 1 : row;
            newRow[rowIndex] = dataTable.Rows[row][column];
        }
        transposed.Rows.Add(newRow);
    }
    return transposed;
}

```

下面进行简单的单元测试：



```
[Fact]
public void Transpose_ColumnNameAsData()
{
    DataTable originalTable = new DataTable("测试");
    originalTable.Columns.Add("A", typeof(string));
    originalTable.Columns.Add("B", typeof(int));
    originalTable.Columns.Add("C", typeof(int));
    originalTable.Rows.Add("D", 1, 2);
    //列名作为数据的情况
    var table = TableHelper.Transpose(originalTable, true);
    Assert.Equal(originalTable.TableName, table.TableName);
    Assert.Equal("Column1", table.Columns[0].ColumnName);
    Assert.Equal("Column2", table.Columns[1].ColumnName);
    Assert.Equal(3, table.Rows.Count);
    Assert.Equal("A", table.Rows[0][0]);
    Assert.Equal("D", table.Rows[0][1]);
    Assert.Equal("B", table.Rows[1][0]);
    Assert.Equal("1", table.Rows[1][1].ToString());
    Assert.Equal("C", table.Rows[2][0]);
    Assert.Equal("2", table.Rows[2][1].ToString());
}

```

***注***：测试方法代码以及示例源码都已经上传至代码库，有兴趣的可以看看。[https://gitee.com/hugogoos/Ideal](https://github.com)


 本博客参考[slowerssr加速器](https://slowerss.com)。转载请注明出处！
