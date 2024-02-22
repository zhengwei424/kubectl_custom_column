# kubectl_custom_column 

实现方式是参考kubectl get 命令的-L参数的终端输出逻辑，通过在源码中添加自定义lable来实现自定义字段打印的效果，感觉这种方式的输出效率最高，如果将自定义字段插入到已有字段中间，会涉及到每行数据的顺序调整，查询量较大时计算量较大，输出慢

/opt/kubernetes-1.15.3/pkg/printers/tableprinter.go
```go
func (h *HumanReadablePrinter) PrintObj(obj runtime.Object, output io.Writer) error {
	w, found := output.(*tabwriter.Writer)
	if !found {
		w = GetNewTabWriter(output)
		output = w
		defer w.Flush()
	}

	// Case 1: Parameter "obj" is a table from server; print it.
	// display tables following the rules of options
	if table, ok := obj.(*metav1beta1.Table); ok {
		// Do not print headers if this table has no column definitions, or they are the same as the last ones we printed
		localOptions := h.options
		if len(table.ColumnDefinitions) == 0 || reflect.DeepEqual(table.ColumnDefinitions, h.lastColumns) {
			localOptions.NoHeaders = true
		}

		if len(table.ColumnDefinitions) == 0 {
			// If this table has no column definitions, use the columns from the last table we printed for decoration and layout.
			// This is done when receiving tables in watch events to save bandwidth.
			localOptions.NoHeaders = true
			table.ColumnDefinitions = h.lastColumns
		} else {
			// If this table has column definitions, remember them for future use.
			h.lastColumns = table.ColumnDefinitions
		}
		// +++++++++++++++++++++++++++++++++++++mycode+++++++++++++++++++++++++++++++++++++++++++++++
		// 添加代码（使用-L的方式添加自定义字段，这样避免对slice重组，提高打印效率）
		// 判断资源类型是否需要添加自定义字段
		customColumns := make([]string, 0)

		if mypkg.IsShowProject(localOptions.Kind.Kind) && !mypkg.IsSkip {
			customColumns = append(customColumns, "project")
			if mypkg.IsShowApp(localOptions.Kind.Kind) {
				customColumns = append(customColumns, "app")
			}
		}
		localOptions.ColumnLabels = append(customColumns, localOptions.ColumnLabels...)
		// +++++++++++++++++++++++++++++++++++++mycode+++++++++++++++++++++++++++++++++++++++++++++++
		// 终端打印代码
		if err := decorateTable(table, localOptions); err != nil {
			return err
		}
		return printTable(table, output, localOptions)
	}


// 在这里添加新的自定义字段,用添加便签的方式，模拟-L参数实现自定义字段
func decorateTable(table *metav1beta1.Table, options PrintOptions) error {
	width := len(table.ColumnDefinitions) + len(options.ColumnLabels)
	if options.WithNamespace {
		width++
	}
	if options.ShowLabels {
		width++
	}

	columns := table.ColumnDefinitions

	nameColumn := -1
	if options.WithKind && !options.Kind.Empty() {
		for i := range columns {
			if columns[i].Format == "name" && columns[i].Type == "string" {
				nameColumn = i
				break
			}
		}
	}

	if width != len(table.ColumnDefinitions) {
		columns = make([]metav1beta1.TableColumnDefinition, 0, width)
		if options.WithNamespace {
			columns = append(columns, metav1beta1.TableColumnDefinition{
				Name: "Namespace",
				Type: "string",
			})
		}
		columns = append(columns, table.ColumnDefinitions...)
		for _, label := range formatLabelHeaders(options.ColumnLabels) {
			columns = append(columns, metav1beta1.TableColumnDefinition{
				Name: label,
				Type: "string",
			})
		}
		if options.ShowLabels {
			columns = append(columns, metav1beta1.TableColumnDefinition{
				Name: "Labels",
				Type: "string",
			})
		}
	}

	rows := table.Rows

	includeLabels := len(options.ColumnLabels) > 0 || options.ShowLabels
	if includeLabels || options.WithNamespace || nameColumn != -1 {
		for i := range rows {
			row := rows[i]

			if nameColumn != -1 {
				row.Cells[nameColumn] = fmt.Sprintf("%s/%s", strings.ToLower(options.Kind.String()), row.Cells[nameColumn])
			}

			var m metav1.Object
			if obj := row.Object.Object; obj != nil {
				if acc, err := meta.Accessor(obj); err == nil {
					m = acc
				}
			}
			// +++++++++++++++++++++++++++++++++++++mycode+++++++++++++++++++++++++++++++++++++++++++++++
			// 获取项目名称并设置lable
			tmpLables := make(map[string]string)
			if len(options.ColumnLabels) > 0 && options.ColumnLabels[0] == "project" {
				if m.GetLabels() != nil {
					tmpLables = m.GetLabels()
				}

				var prj string

				if options.Kind.Kind != "Namespace" {
					prj = mypkg.GetNamespaceFromProject(m.GetNamespace())
				} else {
					prj = mypkg.GetNamespaceFromProject(m.GetName())
				}

				tmpLables["project"] = prj
				m.SetLabels(tmpLables)
			}
			if len(options.ColumnLabels) > 1 && options.ColumnLabels[1] == "app" {
				for k, v := range tmpLables {
					if k == "application_name" || k == "app" {
						tmpLables["app"] = v
						m.SetLabels(tmpLables)
						break
					}
				}
			}
			// +++++++++++++++++++++++++++++++++++++mycode+++++++++++++++++++++++++++++++++++++++++++++++

			// if we can't get an accessor, fill out the appropriate columns with empty spaces
			if m == nil {
				if options.WithNamespace {
					r := make([]interface{}, 1, width)
					row.Cells = append(r, row.Cells...)
				}
				for j := 0; j < width-len(row.Cells); j++ {
					row.Cells = append(row.Cells, nil)
				}
				rows[i] = row
				continue
			}

			if options.WithNamespace {
				r := make([]interface{}, 1, width)
				r[0] = m.GetNamespace()
				row.Cells = append(r, row.Cells...)
			}
			if includeLabels {
				row.Cells = appendLabelCells(row.Cells, m.GetLabels(), options)
			}
			rows[i] = row
		}
	}

	table.ColumnDefinitions = columns
	table.Rows = rows
	return nil
}
```
# 记录一下table的抬头字段入口位置(忽略)
```bash
/opt/kubernetes-1.15.3/pkg/kubectl/cmd/get/humanreadable_flags.go
line 99:	printersinternal.AddHandlers(p) 
```

# 解决tabwriter中英文输出列不对其问题（需要修改format.go及tabwriter.go源码）
/root/sdk/go1.19.1/src/fmt/format.go
```go
// +++++++++++++++++++++++++++++++mycode+++++++++++++++++++++
func stringWidth(s string) int {
	width := 0
	for _, runeValue := range s {
		if utf8.RuneLen(runeValue) == 3 { // 中文字符占用 3 个字节
			width += 2
		} else {
			width += 1
		}
	}
	return width
}
// +++++++++++++++++++++++++++++++mycode+++++++++++++++++++++

// padString appends s to f.buf, padded on left (!f.minus) or right (f.minus).
func (f *fmt) padString(s string) {
	if !f.widPresent || f.wid == 0 {
		f.buf.writeString(s)
		return
	}
    // +++++++++++++++++++++++++++++++mycode+++++++++++++++++++++
	//width := f.wid - utf8.RuneCountInString(s)
	width := f.wid - stringWidth(s)
    // +++++++++++++++++++++++++++++++mycode+++++++++++++++++++++
	if !f.minus {
		// left padding
		f.writePadding(width)
		f.buf.writeString(s)
	} else {
		// right padding
		f.buf.writeString(s)
		f.writePadding(width)
	}
}
```

/root/sdk/go1.19.1/pkg/mod/github.com/liggitt/tabwriter@v0.0.0-20181228230101-89fcab3d43de/tabwriter.go
```go
// ++++++++++++++++++++++mycode+++++++++++++++++++++++++++++++++
func stringWidth(s string) int {
	width := 0
	for _, runeValue := range s {
		if utf8.RuneLen(runeValue) == 3 { // 中文字符占用 3 个字节
			width += 2
		} else {
			width += 1
		}
	}
	return width
}

// ++++++++++++++++++++++mycode+++++++++++++++++++++++++++++++++

// Update the cell width.
func (b *Writer) updateWidth() {
    // ++++++++++++++++++++++mycode+++++++++++++++++++++++++++++++++
	//b.cell.width += utf8.RuneCount(b.buf[b.pos:])
	b.cell.width += stringWidth(string(b.buf[b.pos:]))
	// ++++++++++++++++++++++mycode+++++++++++++++++++++++++++++++++
	b.pos = len(b.buf)
}
```
