The Ultimate Ingress Architecture
Python

# ═══ Production Graph Ingress Pipeline ═══
# multiple tools; different strengths



"""
ARCHITECTURE:

    Raw Sources                    
    ├── Kafka topics ──────────┐   
    ├── PostgreSQL CDC ────────┤   
    ├── S3 Parquet files ──────┤   
    └── REST API events ───────┤   
                               │   
                               ▼   
                    ┌──────────────────┐
                    │  Apache Flink /  │  ← Stream processing
                    │  Spark Structured│     - Entity resolution
                    │  Streaming       │     - Deduplication
                    │                  │     - Schema validation
                    │  OR: Python      │     - Type inference
                    │  service with    │     - Edge aggregation
                    │  pandas/polars   │
                    └───────┬──────────┘
                            │
                    ┌───────┴──────────┐
                    │                  │
                    ▼                  ▼
            ┌──────────────┐  ┌──────────────────┐
            │  FalkorDB /  │  │  Parquet files    │
            │  Kùzu /      │  │  (for Graphistry  │
            │  Neo4j       │  │   direct ingest)  │
            │  (real-time  │  │                   │
            │   queries)   │  │  (batch analysis) │
            └──────┬───────┘  └────────┬──────────┘
                   │                   │
                   └─────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │   GRAPHISTRY     │
                    │   (visualization │
                    │    & investigation│
                    │    layer)        │
                    └──────────────────┘
"""


```python
# ═══ Component 1: Entity Resolution ═══
# The HARDEST part of ingress

import pandas as pd
from collections import defaultdict
import hashlib

class GraphEntityResolver:
    """
    Resolves entities across multiple data sources
    into a canonical graph representation.
    """
    
    def __init__(self):
        self.entity_map = {}       # raw_id → canonical_id
        self.entity_attrs = {}     # canonical_id → merged attributes
        self.resolution_log = []   # audit trail
    
    def add_entity(self, raw_id, source, entity_type, attributes):
        """Add or merge an entity."""
        # Strategy 1: Exact match on normalized key
        canonical = self._normalize(raw_id, entity_type)
        
        if canonical in self.entity_attrs:
            # Merge attributes (source-aware)
            existing = self.entity_attrs[canonical]
            for key, value in attributes.items():
                if key not in existing or existing[key] is None:
                    existing[key] = value
                elif existing[key] != value:
                    # Conflict resolution: keep most recent, log conflict
                    existing[f'{key}_conflict_{source}'] = value
                    self.resolution_log.append({
                        'entity': canonical,
                        'field': key,
                        'old_value': existing[key],
                        'new_value': value,
                        'source': source
                    })
        else:
            self.entity_attrs[canonical] = {
                **attributes,
                '_type': entity_type,
                '_sources': {source},
                '_raw_ids': {raw_id}
            }
        
        self.entity_map[f'{source}:{raw_id}'] = canonical
        self.entity_attrs[canonical].setdefault('_sources', set()).add(source)
        self.entity_attrs[canonical].setdefault('_raw_ids', set()).add(raw_id)
        
        return canonical
    
    def _normalize(self, raw_id, entity_type):
        """Normalize entity ID to canonical form."""
        # Email normalization
        if entity_type == 'email':
            return raw_id.lower().strip()
        
        # Name normalization
        if entity_type == 'person':
            parts = raw_id.lower().strip().split()
            return ' '.join(sorted(parts))  # "John Smith" == "Smith John"
        
        # IP address normalization
        if entity_type == 'ip':
            return raw_id.strip()
        
        # Default: lowercase strip
        return raw_id.lower().strip()
    
    def resolve_edge(self, src_raw, src_source, dst_raw, dst_source,
                     edge_type, edge_attrs):
        """Resolve an edge's endpoints and return canonical edge."""
        src_canonical = self.entity_map.get(f'{src_source}:{src_raw}')
        dst_canonical = self.entity_map.get(f'{dst_source}:{dst_raw}')
        
        if src_canonical is None or dst_canonical is None:
            return None  # Unresolved entity
        
        return {
            'src': src_canonical,
            'dst': dst_canonical,
            'type': edge_type,
            **edge_attrs
        }
    
    def to_dataframes(self):
        """Export resolved entities and edges to DataFrames."""
        nodes = []
        for canonical_id, attrs in self.entity_attrs.items():
            row = {'id': canonical_id}
            for k, v in attrs.items():
                if isinstance(v, set):
                    row[k] = ','.join(str(x) for x in v)
                else:
                    row[k] = v
            nodes.append(row)
        
        return pd.DataFrame(nodes)


# ═══ Component 2: Multi-Source Ingestion ═══

class GraphIngestor:
    """
    Ingest data from multiple sources into a unified graph.
    """
    
    def __init__(self, resolver=None):
        self.resolver = resolver or GraphEntityResolver()
        self.edges = []
    
    def ingest_csv_entities(self, filepath, id_col, type_col=None, 
                            entity_type='generic', source='csv'):
        """Ingest entities from CSV."""
        df = pd.read_csv(filepath)
        for _, row in df.iterrows():
            etype = row[type_col] if type_col else entity_type
            attrs = {k: v for k, v in row.items() 
                     if k not in [id_col, type_col]}
            self.resolver.add_entity(
                raw_id=str(row[id_col]),
                source=source,
                entity_type=etype,
                attributes=attrs
            )
    
    def ingest_csv_edges(self, filepath, src_col, dst_col,
                          edge_type_col=None, edge_type='related',
                          source='csv'):
        """Ingest edges from CSV."""
        df = pd.read_csv(filepath)
        for _, row in df.iterrows():
            etype = row[edge_type_col] if edge_type_col else edge_type
            attrs = {k: v for k, v in row.items()
                     if k not in [src_col, dst_col, edge_type_col]}
            
            edge = self.resolver.resolve_edge(
                src_raw=str(row[src_col]), src_source=source,
                dst_raw=str(row[dst_col]), dst_source=source,
                edge_type=etype, edge_attrs=attrs
            )
            if edge:
                self.edges.append(edge)
    
    def ingest_from_sql(self, engine, query, src_col, dst_col,
                         source='sql'):
        """Ingest from SQL query result."""
        df = pd.read_sql(query, engine)
        # Auto-register entities
        for col in [src_col, dst_col]:
            for val in df[col].unique():
                self.resolver.add_entity(
                    raw_id=str(val), source=source,
                    entity_type='entity', attributes={}
                )
        
        for _, row in df.iterrows():
            attrs = {k: v for k, v in row.items()
                     if k not in [src_col, dst_col]}
            edge = self.resolver.resolve_edge(
                src_raw=str(row[src_col]), src_source=source,
                dst_raw=str(row[dst_col]), dst_source=source,
                edge_type='relationship', edge_attrs=attrs
            )
            if edge:
                self.edges.append(edge)
    
    def ingest_from_kafka(self, topic, bootstrap_servers,
                           parse_fn, max_messages=10000):
        """Ingest from Kafka topic."""
        from kafka import KafkaConsumer
        import json
        
        consumer = KafkaConsumer(
            topic,
            bootstrap_servers=bootstrap_servers,
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
            auto_offset_reset='earliest',
            consumer_timeout_ms=5000
        )
        
        count = 0
        for message in consumer:
            if count >= max_messages:
                break
            
            # parse_fn should return:
            # {'entities': [...], 'edges': [...]}
            parsed = parse_fn(message.value)
            
            for entity in parsed.get('entities', []):
                self.resolver.add_entity(**entity)
            
            for edge in parsed.get('edges', []):
                resolved = self.resolver.resolve_edge(**edge)
                if resolved:
                    self.edges.append(resolved)
            
            count += 1
        
        consumer.close()
    
    def to_graphistry(self):
        """Export to Graphistry visualization."""
        nodes_df = self.resolver.to_dataframes()
        edges_df = pd.DataFrame(self.edges)
        
        if len(edges_df) == 0:
            raise ValueError("No edges ingested")
        
        g = (graphistry
            .edges(edges_df, 'src', 'dst')
            .nodes(nodes_df, 'id')
        )
        
        if '_type' in nodes_df.columns:
            g = g.bind(point_color='_type')
        if 'type' in edges_df.columns:
            g = g.bind(edge_color='type')
        
        return g
    
    def to_falkordb(self, host='localhost', port=6379, graph_name='graph'):
        """Export to FalkorDB."""
        from falkordb import FalkorDB
        
        db = FalkorDB(host=host, port=port)
        graph = db.select_graph(graph_name)
        
        nodes_df = self.resolver.to_dataframes()
        edges_df = pd.DataFrame(self.edges)
        
        # Create nodes
        for _, row in nodes_df.iterrows():
            props = {k: v for k, v in row.items() 
                     if k != 'id' and not k.startswith('_')
                     and pd.notna(v)}
            props_str = ', '.join(f'{k}: "{v}"' if isinstance(v, str) 
                                  else f'{k}: {v}'
                                  for k, v in props.items())
            node_type = row.get('_type', 'Entity')
            if isinstance(node_type, str):
                node_type = node_type.replace(' ', '_')
            
            query = f'CREATE (:{node_type} {{name: "{row["id"]}", {props_str}}})'
            try:
                graph.query(query)
            except Exception as e:
                print(f"Warning: {e}")
        
        # Create edges
        for _, row in edges_df.iterrows():
            rel_type = row.get('type', 'RELATED').upper().replace(' ', '_')
            props = {k: v for k, v in row.items()
                     if k not in ['src', 'dst', 'type']
                     and pd.notna(v)}
            props_str = ', '.join(f'{k}: "{v}"' if isinstance(v, str)
                                  else f'{k}: {v}'
                                  for k, v in props.items())
            
            query = f"""
                MATCH (a {{name: "{row['src']}"}}), (b {{name: "{row['dst']}"}})
                CREATE (a)-[:{rel_type} {{{props_str}}}]->(b)
            """
            try:
                graph.query(query)
            except Exception as e:
                print(f"Warning: {e}")
        
        return graph
    
    def to_kuzu(self, db_path='./kuzu_db'):
        """Export to Kùzu embedded database."""
        import kuzu
        
        db = kuzu.Database(db_path)
        conn = kuzu.Connection(db)
        
        nodes_df = self.resolver.to_dataframes()
        edges_df = pd.DataFrame(self.edges)
        
        # Kùzu needs explicit schema — create tables based on data
        node_types = nodes_df['_type'].unique() if '_type' in nodes_df.columns else ['Entity']
        
        for ntype in node_types:
            try:
                conn.execute(f"""
                    CREATE NODE TABLE {ntype}(
                        name STRING, 
                        PRIMARY KEY(name)
                    )
                """)
            except Exception:
                pass  # Table might already exist
        
        # Load nodes
        for _, row in nodes_df.iterrows():
            ntype = row.get('_type', 'Entity')
            try:
                conn.execute(f"CREATE (:{ntype} {{name: '{row['id']}'}})")
            except Exception:
                pass
        
        return conn
```


# ═══ Usage Example ═══

# ingestor = GraphIngestor()
#
# # Ingest from multiple sources
# ingestor.ingest_csv_entities('employees.csv', 'emp_id', entity_type='person', source='hr')
# ingestor.ingest_csv_entities('accounts.csv', 'account_id', entity_type='account', source='finance')
# ingestor.ingest_csv_edges('transactions.csv', 'from_acct', 'to_acct', source='finance')
# ingestor.ingest_csv_edges('communications.csv', 'sender', 'recipient', source='email')
#
# # Export to visualization
# g = ingestor.to_graphistry()
# g.plot()
#
# # Export to FalkorDB for real-time querying
# graph = ingestor.to_falkordb(graph_name='investigation')
#
# # Export to Kùzu for analytical queries
# conn = ingestor.to_kuzu('./investigation_db')


# The Performance Stack: FalkorDB + cuGraph + Graphistry
The bleeding-edge performance stack for graph analytics.
