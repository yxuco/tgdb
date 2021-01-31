# TIBCO Graph DB client API

Forked from <https://github.com/TIBCOSoftware/tgdb-client/tree/master/api/go/src/tgdb>, and modified `imports` to support Go modules.

## Example

```
package main

import (
	"fmt"
	"os"

	"github.com/yxuco/tgdb"
	"github.com/yxuco/tgdb/factory"
	"github.com/yxuco/tgdb/impl"
)

func main() {
	url := "tcp://127.0.0.1:8222/{dbName=housedb}"
	user := "napoleon"
	passwd := "bonaparte"

	cf := factory.GetConnectionFactory()
	conn, err := cf.CreateAdminConnection(url, user, passwd, nil)
	if err != nil {
		fmt.Printf("connection error: %s, %s\n", err.GetErrorCode(), err.GetErrorMsg())
		os.Exit(1)
	}
	conn.Connect()
	//defer conn.Disconnect()

	searchAndUpdate(conn)
	execQuery(conn)
	edgeQuery(conn)
}

func searchAndUpdate(conn tgdb.TGConnection) {
	memberName := "Napoleon Bonaparte"
	fmt.Printf("\n*** searchGraph %s\n", memberName)

	gof, err := conn.GetGraphObjectFactory()
	if err != nil {
		fmt.Printf("graph object factory error: %s, %s\n", err.GetErrorCode(), err.GetErrorMsg())
		return
	}
	_, err = conn.GetGraphMetadata(true)

	key, err := gof.CreateCompositeKey("houseMemberType")
	key.SetOrCreateAttribute("memberName", memberName)
	fmt.Printf("search house member: %s\n", memberName)
	opts := impl.NewQueryOption()
	opts.SetEdgeLimit(0)
	opts.SetTraversalDepth(3)
	member, err := conn.GetEntity(key, opts)
	if err != nil {
		fmt.Printf("Failed to fetch member: %v\n", err)
	}
	if member != nil {
		if node, ok := member.(tgdb.TGNode); ok {
			edges := node.GetEdges()
			fmt.Printf("check relationships: %d\n", len(edges))
			// this does not work becaue above call always returns 0 edge
			for _, edge := range edges {
				n := edge.GetVertices()
				fmt.Printf("relationship '%v': %v -> %v\n", edge.GetAttribute("relType").GetValue(),
					n[0].GetAttribute("memberName").GetValue(), n[1].GetAttribute("memberName").GetValue())
			}
		}
		if attrs, err := member.GetAttributes(); err == nil {
			for _, v := range attrs {
				fmt.Printf("\tattribute %s => %v\n", v.GetName(), v.GetValue())
			}
		}

		// test update entity
		yrBorn := member.GetAttribute("yearBorn")
		yrBorn.SetValue(yrBorn.GetValue().(int) - 1)
		v := member.GetAttribute("yearBorn")
		fmt.Printf("\tupdated %s => %v\n", v.GetName(), v.GetValue())
		conn.UpdateEntity(member)
		conn.Commit()
		v = member.GetAttribute("yearBorn")
		fmt.Printf("\tcommitted %s => %v\n", v.GetName(), v.GetValue())
	}
}

func execQuery(conn tgdb.TGConnection) {
	startYear := 1800
	endYear := 1900
	fmt.Printf("\n*** execQuery born between (%d, %d)\n", startYear, endYear)

	query := fmt.Sprintf("gremlin://g.V().has('houseMemberType', 'yearBorn', between(%d, %d));", startYear, endYear)
	rset, err := conn.ExecuteQuery(query, nil)
	if err != nil {
		fmt.Printf("query error: %v\n", err)
	}
	for rset.HasNext() {
		if member, ok := rset.Next().(tgdb.TGNode); ok {
			fmt.Printf("Found member %v\n", member.GetAttribute("memberName").GetValue())
			if attrs, err := member.GetAttributes(); err == nil {
				for _, v := range attrs {
					fmt.Printf("\tattribute %s => %v\n", v.GetName(), v.GetValue())
				}
			}
		}
	}
}

func edgeQuery(conn tgdb.TGConnection) {
	memberName := "Napoleon Bonaparte"
	fmt.Printf("\n*** edgeQuery %s\n", memberName)
	query := fmt.Sprintf("gremlin://g.V().has('houseMemberType', 'memberName', '%s').bothE();", memberName)
	rset, err := conn.ExecuteQuery(query, nil)
	if err != nil {
		fmt.Printf("query error: %v\n", err)
	}
	for rset.HasNext() {
		edge := rset.Next().(tgdb.TGEdge)
		fmt.Println("Edge")
		attrs, _ := edge.GetAttributes()
		for _, a := range attrs {
			fmt.Printf("\tattribute %s -> %v\n", a.GetName(), a.GetValue())
		}
		n := edge.GetVertices()
		for i, v := range n {
			fmt.Printf("\tnode %d: %d\n", i, v.GetVirtualId())
		}
	}
}
```
