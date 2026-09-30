# Part 27: Search with Elasticsearch

## ภาพรวม

Elasticsearch คือ distributed search และ analytics engine ที่สร้างบน Apache Lucene ใช้สำหรับ full-text search, log analytics, และ real-time data analysis ในระบบ Microservices เราใช้ Elasticsearch เป็น dedicated search service ที่ sync ข้อมูลจาก primary database (PostgreSQL) แล้วให้ search capabilities ที่ซับซ้อนกว่าที่ SQL จะทำได้

### สิ่งที่จะเรียนในบทนี้

- Elasticsearch setup และ index mappings
- Full-text search พร้อม Thai language analyzer
- Multi-field search with boosting
- Faceted search / aggregations
- Search suggestions (autocomplete)
- Bulk indexing สำหรับ performance
- Index lifecycle management (ILM)
- Sync ข้อมูลจาก PostgreSQL → Elasticsearch
- Search service class ใน Node.js
- Docker Compose setup

---

## 1. Docker Compose Setup

```yaml
# docker-compose.yml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: elasticsearch
    environment:
      - node.name=elasticsearch
      - cluster.name=microservices-cluster
      - discovery.type=single-node
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
      - xpack.security.enabled=false
      - xpack.security.enrollment.enabled=false
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
      - "9300:9300"
    networks:
      - search-network
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_cluster/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - xpack.security.enabled=false
    ports:
      - "5601:5601"
    depends_on:
      elasticsearch:
        condition: service_healthy
    networks:
      - search-network

  # ICU Analysis Plugin installer (run once)
  elasticsearch-setup:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: elasticsearch-setup
    depends_on:
      elasticsearch:
        condition: service_healthy
    networks:
      - search-network
    command: >
      bash -c "
        elasticsearch-plugin install analysis-icu --batch &&
        elasticsearch-plugin install analysis-smartcn --batch
      "
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data

  search-service:
    build:
      context: ./search-service
      dockerfile: Dockerfile
    container_name: search-service
    environment:
      - NODE_ENV=production
      - ELASTICSEARCH_URL=http://elasticsearch:9200
      - DATABASE_URL=postgresql://user:password@postgres:5432/products_db
      - PORT=3007
    ports:
      - "3007:3007"
    depends_on:
      elasticsearch:
        condition: service_healthy
    networks:
      - search-network
      - microservices-network

  postgres:
    image: postgres:15-alpine
    container_name: postgres-products
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=products_db
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"
    networks:
      - microservices-network

volumes:
  elasticsearch_data:
  postgres_data:

networks:
  search-network:
    driver: bridge
  microservices-network:
    driver: bridge
```

---

## 2. Index Mapping พร้อม Thai Language Support

```javascript
// src/mappings/product.mapping.js

const PRODUCT_INDEX_MAPPING = {
  settings: {
    number_of_shards: 1,
    number_of_replicas: 0,
    analysis: {
      // Custom analyzers
      analyzer: {
        // Thai language analyzer
        thai_analyzer: {
          type: 'custom',
          tokenizer: 'thai',
          filter: ['lowercase', 'thai_stop', 'thai_synonym']
        },
        // Thai + ICU analyzer สำหรับ better tokenization
        thai_icu_analyzer: {
          type: 'custom',
          tokenizer: 'icu_tokenizer',
          filter: ['icu_folding', 'lowercase', 'thai_stop']
        },
        // English analyzer สำหรับ product names
        english_analyzer: {
          type: 'custom',
          tokenizer: 'standard',
          filter: ['lowercase', 'english_stop', 'english_stemmer']
        },
        // Search-time analyzer
        thai_search_analyzer: {
          type: 'custom',
          tokenizer: 'thai',
          filter: ['lowercase', 'thai_stop', 'thai_synonym']
        },
        // Autocomplete analyzer
        autocomplete_analyzer: {
          type: 'custom',
          tokenizer: 'autocomplete_tokenizer',
          filter: ['lowercase']
        },
        autocomplete_search_analyzer: {
          type: 'custom',
          tokenizer: 'keyword',
          filter: ['lowercase']
        }
      },
      // Custom tokenizers
      tokenizer: {
        autocomplete_tokenizer: {
          type: 'edge_ngram',
          min_gram: 2,
          max_gram: 20,
          token_chars: ['letter', 'digit']
        }
      },
      // Custom filters
      filter: {
        thai_stop: {
          type: 'stop',
          stopwords: ['และ', 'หรือ', 'ของ', 'ใน', 'ที่', 'การ', 'ได้', 'จาก', 'มี', 'เป็น', 'กับ', 'ให้']
        },
        thai_synonym: {
          type: 'synonym',
          synonyms: [
            'มือถือ,โทรศัพท์มือถือ,สมาร์ทโฟน',
            'แล็ปท็อป,โน้ตบุ๊ค,คอมพิวเตอร์พกพา',
            'ทีวี,โทรทัศน์,จอทีวี'
          ]
        },
        english_stop: {
          type: 'stop',
          stopwords: '_english_'
        },
        english_stemmer: {
          type: 'stemmer',
          language: 'english'
        }
      }
    }
  },
  mappings: {
    dynamic: false,
    properties: {
      id: { type: 'keyword' },
      sku: { type: 'keyword' },
      
      // Product name - multi-field สำหรับ different search strategies
      name: {
        type: 'text',
        analyzer: 'thai_analyzer',
        search_analyzer: 'thai_search_analyzer',
        fields: {
          english: {
            type: 'text',
            analyzer: 'english_analyzer'
          },
          autocomplete: {
            type: 'text',
            analyzer: 'autocomplete_analyzer',
            search_analyzer: 'autocomplete_search_analyzer'
          },
          keyword: {
            type: 'keyword',
            ignore_above: 256
          }
        }
      },

      // Description - Thai text
      description: {
        type: 'text',
        analyzer: 'thai_analyzer',
        search_analyzer: 'thai_search_analyzer'
      },

      // Short description สำหรับ snippet
      shortDescription: {
        type: 'text',
        analyzer: 'thai_analyzer'
      },

      // Price fields
      price: {
        type: 'double'
      },
      originalPrice: {
        type: 'double'
      },
      discountPercent: {
        type: 'integer'
      },

      // Category hierarchy
      category: {
        type: 'object',
        properties: {
          id: { type: 'keyword' },
          name: {
            type: 'text',
            analyzer: 'thai_analyzer',
            fields: {
              keyword: { type: 'keyword' }
            }
          },
          path: { type: 'keyword' }, // เช่น "electronics/mobile/smartphone"
          ancestors: {
            type: 'nested',
            properties: {
              id: { type: 'keyword' },
              name: { type: 'keyword' }
            }
          }
        }
      },

      // Brand
      brand: {
        type: 'object',
        properties: {
          id: { type: 'keyword' },
          name: {
            type: 'text',
            fields: {
              keyword: { type: 'keyword' }
            }
          }
        }
      },

      // Tags
      tags: {
        type: 'keyword'
      },

      // Attributes สำหรับ faceted search
      attributes: {
        type: 'nested',
        properties: {
          key: { type: 'keyword' },
          value: { type: 'keyword' },
          displayName: {
            type: 'text',
            fields: { keyword: { type: 'keyword' } }
          }
        }
      },

      // Metrics สำหรับ ranking
      metrics: {
        type: 'object',
        properties: {
          viewCount: { type: 'long' },
          salesCount: { type: 'long' },
          rating: { type: 'float' },
          reviewCount: { type: 'integer' },
          wishlistCount: { type: 'integer' }
        }
      },

      // Stock & availability
      inStock: { type: 'boolean' },
      stockQuantity: { type: 'integer' },

      // Status
      status: { type: 'keyword' }, // active, inactive, draft

      // Timestamps
      createdAt: { type: 'date' },
      updatedAt: { type: 'date' },

      // Location สำหรับ geo search
      location: { type: 'geo_point' },

      // Suggest field สำหรับ completion suggester
      suggest: {
        type: 'completion',
        analyzer: 'thai_analyzer',
        search_analyzer: 'thai_search_analyzer'
      }
    }
  }
};

module.exports = PRODUCT_INDEX_MAPPING;
```

---

## 3. Elasticsearch Client Setup

```javascript
// src/client/elasticsearch.client.js

const { Client } = require('@elastic/elasticsearch');

class ElasticsearchClient {
  constructor() {
    this.client = null;
    this.isConnected = false;
  }

  async connect() {
    const config = {
      node: process.env.ELASTICSEARCH_URL || 'http://localhost:9200',
      maxRetries: 5,
      requestTimeout: 30000,
      sniffOnStart: false,
      // สำหรับ production ให้ใส่ auth
      ...(process.env.ELASTICSEARCH_USERNAME && {
        auth: {
          username: process.env.ELASTICSEARCH_USERNAME,
          password: process.env.ELASTICSEARCH_PASSWORD
        }
      })
    };

    this.client = new Client(config);

    // Test connection
    try {
      const info = await this.client.info();
      console.log(`Connected to Elasticsearch ${info.version.number}`);
      this.isConnected = true;
    } catch (error) {
      console.error('Failed to connect to Elasticsearch:', error.message);
      throw error;
    }

    return this.client;
  }

  getClient() {
    if (!this.isConnected) {
      throw new Error('Elasticsearch client not connected. Call connect() first.');
    }
    return this.client;
  }

  async disconnect() {
    if (this.client) {
      await this.client.close();
      this.isConnected = false;
    }
  }
}

// Singleton instance
const esClient = new ElasticsearchClient();

module.exports = esClient;
```

---

## 4. Index Management

```javascript
// src/services/index.manager.js

const esClient = require('../client/elasticsearch.client');
const PRODUCT_INDEX_MAPPING = require('../mappings/product.mapping');

const INDEX_NAME = 'products';
const INDEX_ALIAS = 'products_alias';

class IndexManager {
  constructor() {
    this.client = null;
  }

  async initialize() {
    this.client = esClient.getClient();
  }

  /**
   * Create index พร้อม mapping
   */
  async createIndex(indexName = INDEX_NAME) {
    const exists = await this.client.indices.exists({ index: indexName });
    
    if (exists) {
      console.log(`Index ${indexName} already exists`);
      return false;
    }

    await this.client.indices.create({
      index: indexName,
      body: PRODUCT_INDEX_MAPPING
    });

    console.log(`Index ${indexName} created successfully`);

    // Create alias
    await this.client.indices.putAlias({
      index: indexName,
      name: INDEX_ALIAS
    });

    return true;
  }

  /**
   * Zero-downtime reindex
   * 1. สร้าง new index version
   * 2. Reindex ข้อมูลไป new index
   * 3. สลับ alias
   * 4. ลบ old index
   */
  async reindex() {
    const timestamp = Date.now();
    const newIndexName = `products_${timestamp}`;

    console.log(`Starting reindex to ${newIndexName}...`);

    // 1. Create new index
    await this.client.indices.create({
      index: newIndexName,
      body: PRODUCT_INDEX_MAPPING
    });

    // 2. Reindex data
    const reindexResult = await this.client.reindex({
      body: {
        source: { index: INDEX_ALIAS },
        dest: { index: newIndexName }
      },
      wait_for_completion: true,
      requests_per_second: 500  // Throttle เพื่อไม่กระทบ production
    });

    console.log(`Reindexed ${reindexResult.total} documents`);

    // 3. Get current index pointing to alias
    const aliasInfo = await this.client.indices.getAlias({ name: INDEX_ALIAS });
    const oldIndexNames = Object.keys(aliasInfo);

    // 4. Atomic alias swap
    const actions = [
      { add: { index: newIndexName, alias: INDEX_ALIAS } },
      ...oldIndexNames.map(name => ({ remove: { index: name, alias: INDEX_ALIAS } }))
    ];

    await this.client.indices.updateAliases({ body: { actions } });

    // 5. Delete old indices
    if (oldIndexNames.length > 0) {
      await this.client.indices.delete({ index: oldIndexNames });
      console.log(`Deleted old indices: ${oldIndexNames.join(', ')}`);
    }

    return newIndexName;
  }

  /**
   * Update mapping (เพิ่ม fields ใหม่ได้ แต่แก้ existing fields ไม่ได้)
   */
  async updateMapping(newFields) {
    await this.client.indices.putMapping({
      index: INDEX_ALIAS,
      body: {
        properties: newFields
      }
    });
  }

  /**
   * Index stats
   */
  async getStats() {
    const stats = await this.client.indices.stats({ index: INDEX_ALIAS });
    const indexStats = stats.indices;
    
    return {
      documentCount: stats._all.total.docs.count,
      deletedDocs: stats._all.total.docs.deleted,
      storeSizeBytes: stats._all.total.store.size_in_bytes,
      indexingRate: stats._all.total.indexing.index_total
    };
  }
}

module.exports = new IndexManager();
```

---

## 5. Index Lifecycle Management (ILM)

```javascript
// src/services/ilm.service.js

class ILMService {
  constructor(esClient) {
    this.client = esClient;
  }

  /**
   * Setup ILM policy สำหรับ log indices
   */
  async createLogILMPolicy() {
    await this.client.ilm.putLifecycle({
      name: 'search-logs-policy',
      body: {
        policy: {
          phases: {
            hot: {
              min_age: '0ms',
              actions: {
                rollover: {
                  max_size: '50gb',
                  max_age: '7d',
                  max_docs: 1000000
                },
                set_priority: { priority: 100 }
              }
            },
            warm: {
              min_age: '7d',
              actions: {
                shrink: { number_of_shards: 1 },
                forcemerge: { max_num_segments: 1 },
                set_priority: { priority: 50 }
              }
            },
            cold: {
              min_age: '30d',
              actions: {
                freeze: {},
                set_priority: { priority: 0 }
              }
            },
            delete: {
              min_age: '90d',
              actions: {
                delete: {}
              }
            }
          }
        }
      }
    });

    // Create index template ที่ใช้ policy นี้
    await this.client.indices.putIndexTemplate({
      name: 'search-logs-template',
      body: {
        index_patterns: ['search-logs-*'],
        template: {
          settings: {
            'index.lifecycle.name': 'search-logs-policy',
            'index.lifecycle.rollover_alias': 'search-logs'
          }
        }
      }
    });

    // Create initial index
    try {
      await this.client.indices.create({
        index: 'search-logs-000001',
        body: {
          aliases: {
            'search-logs': { is_write_index: true }
          }
        }
      });
    } catch (error) {
      if (error.meta?.statusCode !== 400) throw error; // Ignore if already exists
    }
  }
}

module.exports = ILMService;
```

---

## 6. Search Service

```javascript
// src/services/search.service.js

const esClient = require('../client/elasticsearch.client');

const INDEX_ALIAS = 'products_alias';

class SearchService {
  constructor() {
    this.client = null;
  }

  initialize() {
    this.client = esClient.getClient();
  }

  /**
   * Full-text search พร้อม boosting
   * @param {Object} params - Search parameters
   */
  async search(params) {
    const {
      query = '',
      page = 1,
      limit = 20,
      sort = 'relevance',
      filters = {},
      facets = true
    } = params;

    const from = (page - 1) * limit;

    const searchBody = {
      from,
      size: limit,
      query: this._buildQuery(query, filters),
      sort: this._buildSort(sort, query),
      highlight: this._buildHighlight(),
      aggs: facets ? this._buildAggregations() : undefined,
      _source: {
        excludes: ['suggest', 'description'] // ไม่ return fields ที่ใหญ่
      }
    };

    const result = await this.client.search({
      index: INDEX_ALIAS,
      body: searchBody
    });

    return this._formatSearchResult(result, page, limit);
  }

  /**
   * Build query with multi-field boosting
   */
  _buildQuery(queryText, filters) {
    const mustClauses = [];
    const filterClauses = [];

    // Full-text query
    if (queryText) {
      mustClauses.push({
        multi_match: {
          query: queryText,
          type: 'best_fields',
          fields: [
            'name^4',              // name มี boost สูงสุด
            'name.english^3',      // English name
            'name.autocomplete^2', // Autocomplete
            'brand.name^3',        // Brand name
            'category.name^2',     // Category
            'tags^2',              // Tags
            'description^1',       // Description ต่ำสุด
            'shortDescription^1.5'
          ],
          fuzziness: 'AUTO',       // Typo tolerance
          prefix_length: 2,        // ต้อง match 2 chars แรกก่อน
          minimum_should_match: '75%',
          tie_breaker: 0.3
        }
      });
    }

    // Status filter (always)
    filterClauses.push({ term: { status: 'active' } });
    filterClauses.push({ term: { inStock: true } });

    // Dynamic filters
    if (filters.categoryId) {
      filterClauses.push({ term: { 'category.id': filters.categoryId } });
    }

    if (filters.brandIds && filters.brandIds.length > 0) {
      filterClauses.push({ terms: { 'brand.id': filters.brandIds } });
    }

    if (filters.priceMin !== undefined || filters.priceMax !== undefined) {
      const rangeFilter = { range: { price: {} } };
      if (filters.priceMin !== undefined) rangeFilter.range.price.gte = filters.priceMin;
      if (filters.priceMax !== undefined) rangeFilter.range.price.lte = filters.priceMax;
      filterClauses.push(rangeFilter);
    }

    if (filters.ratingMin) {
      filterClauses.push({
        range: { 'metrics.rating': { gte: filters.ratingMin } }
      });
    }

    // Nested attribute filter
    if (filters.attributes && Object.keys(filters.attributes).length > 0) {
      for (const [key, values] of Object.entries(filters.attributes)) {
        filterClauses.push({
          nested: {
            path: 'attributes',
            query: {
              bool: {
                must: [
                  { term: { 'attributes.key': key } },
                  { terms: { 'attributes.value': Array.isArray(values) ? values : [values] } }
                ]
              }
            }
          }
        });
      }
    }

    // Boosting query สำหรับ popular products
    const boostQuery = {
      function_score: {
        query: {
          bool: {
            must: mustClauses.length > 0 ? mustClauses : [{ match_all: {} }],
            filter: filterClauses
          }
        },
        functions: [
          {
            field_value_factor: {
              field: 'metrics.salesCount',
              factor: 0.1,
              modifier: 'log1p',
              missing: 0
            },
            weight: 2
          },
          {
            field_value_factor: {
              field: 'metrics.rating',
              factor: 1,
              modifier: 'none',
              missing: 0
            },
            weight: 3
          },
          {
            gauss: {
              createdAt: {
                origin: 'now',
                scale: '30d',  // Products ใหม่ภายใน 30 วัน ได้ boost
                offset: '7d',
                decay: 0.5
              }
            },
            weight: 1
          }
        ],
        boost_mode: 'sum',
        score_mode: 'sum'
      }
    };

    return boostQuery;
  }

  /**
   * Build sort options
   */
  _buildSort(sort, queryText) {
    const sortOptions = {
      relevance: queryText ? ['_score'] : [{ 'metrics.salesCount': { order: 'desc' } }],
      price_asc: [{ price: { order: 'asc' } }],
      price_desc: [{ price: { order: 'desc' } }],
      newest: [{ createdAt: { order: 'desc' } }],
      rating: [{ 'metrics.rating': { order: 'desc' } }],
      popular: [{ 'metrics.salesCount': { order: 'desc' } }]
    };

    return sortOptions[sort] || sortOptions.relevance;
  }

  /**
   * Build highlight configuration
   */
  _buildHighlight() {
    return {
      pre_tags: ['<mark>'],
      post_tags: ['</mark>'],
      number_of_fragments: 3,
      fragment_size: 150,
      fields: {
        name: { number_of_fragments: 1 },
        description: { number_of_fragments: 2 },
        shortDescription: { number_of_fragments: 1 }
      }
    };
  }

  /**
   * Build aggregations สำหรับ faceted search
   */
  _buildAggregations() {
    return {
      // Category facet
      categories: {
        terms: {
          field: 'category.name.keyword',
          size: 20
        }
      },

      // Brand facet
      brands: {
        terms: {
          field: 'brand.name.keyword',
          size: 20
        }
      },

      // Price ranges
      priceRanges: {
        range: {
          field: 'price',
          ranges: [
            { key: 'under-500', to: 500 },
            { key: '500-1000', from: 500, to: 1000 },
            { key: '1000-5000', from: 1000, to: 5000 },
            { key: '5000-10000', from: 5000, to: 10000 },
            { key: 'over-10000', from: 10000 }
          ]
        }
      },

      // Price stats
      priceStats: {
        stats: { field: 'price' }
      },

      // Rating distribution
      ratings: {
        histogram: {
          field: 'metrics.rating',
          interval: 1,
          min_doc_count: 0,
          extended_bounds: { min: 1, max: 5 }
        }
      },

      // Nested attributes facet
      attributes: {
        nested: { path: 'attributes' },
        aggs: {
          keys: {
            terms: { field: 'attributes.key', size: 20 },
            aggs: {
              values: {
                terms: { field: 'attributes.value', size: 50 }
              }
            }
          }
        }
      },

      // Total in-stock count
      inStockCount: {
        filter: { term: { inStock: true } }
      }
    };
  }

  /**
   * Format search result
   */
  _formatSearchResult(result, page, limit) {
    const { hits, aggregations } = result;

    return {
      total: hits.total.value,
      page,
      limit,
      pages: Math.ceil(hits.total.value / limit),
      items: hits.hits.map(hit => ({
        id: hit._id,
        score: hit._score,
        ...hit._source,
        highlight: hit.highlight || {}
      })),
      facets: aggregations ? this._formatFacets(aggregations) : null
    };
  }

  /**
   * Format aggregations เป็น facet format
   */
  _formatFacets(aggregations) {
    const facets = {};

    if (aggregations.categories) {
      facets.categories = aggregations.categories.buckets.map(b => ({
        value: b.key,
        count: b.doc_count
      }));
    }

    if (aggregations.brands) {
      facets.brands = aggregations.brands.buckets.map(b => ({
        value: b.key,
        count: b.doc_count
      }));
    }

    if (aggregations.priceRanges) {
      facets.priceRanges = aggregations.priceRanges.buckets.map(b => ({
        key: b.key,
        from: b.from,
        to: b.to,
        count: b.doc_count
      }));
    }

    if (aggregations.priceStats) {
      facets.priceStats = {
        min: aggregations.priceStats.min,
        max: aggregations.priceStats.max,
        avg: aggregations.priceStats.avg
      };
    }

    if (aggregations.attributes?.keys?.buckets) {
      facets.attributes = {};
      aggregations.attributes.keys.buckets.forEach(keyBucket => {
        facets.attributes[keyBucket.key] = keyBucket.values.buckets.map(v => ({
          value: v.key,
          count: v.doc_count
        }));
      });
    }

    return facets;
  }

  /**
   * Autocomplete / Search suggestions
   */
  async suggest(text, limit = 10) {
    // Completion suggester
    const completionResult = await this.client.search({
      index: INDEX_ALIAS,
      body: {
        suggest: {
          productSuggest: {
            prefix: text,
            completion: {
              field: 'suggest',
              size: limit,
              skip_duplicates: true,
              fuzzy: {
                fuzziness: 1,
                prefix_length: 2
              }
            }
          }
        },
        _source: false
      }
    });

    // Multi-match prefix query สำหรับ fallback
    const prefixResult = await this.client.search({
      index: INDEX_ALIAS,
      body: {
        size: limit,
        query: {
          multi_match: {
            query: text,
            type: 'phrase_prefix',
            fields: ['name^3', 'brand.name^2', 'category.name'],
            max_expansions: 20
          }
        },
        _source: ['name', 'category.name', 'brand.name'],
        filter: { term: { status: 'active' } }
      }
    });

    // Merge results
    const suggestions = new Map();

    completionResult.suggest.productSuggest[0].options.forEach(opt => {
      suggestions.set(opt._source?.name || opt.text, {
        text: opt.text,
        score: opt._score,
        type: 'completion'
      });
    });

    prefixResult.hits.hits.forEach(hit => {
      if (!suggestions.has(hit._source.name)) {
        suggestions.set(hit._source.name, {
          text: hit._source.name,
          category: hit._source.category?.name,
          brand: hit._source.brand?.name,
          score: hit._score,
          type: 'prefix'
        });
      }
    });

    return Array.from(suggestions.values()).slice(0, limit);
  }

  /**
   * "Did you mean" spell correction
   */
  async didYouMean(text) {
    const result = await this.client.search({
      index: INDEX_ALIAS,
      body: {
        size: 0,
        suggest: {
          spellCheck: {
            text,
            term: {
              field: 'name',
              suggest_mode: 'missing',
              min_word_length: 3,
              max_edits: 2,
              prefix_length: 2,
              sort: 'score'
            }
          }
        }
      }
    });

    const options = result.suggest.spellCheck;
    if (!options || options.length === 0) return null;

    // รวม suggestions เป็น phrase
    const corrected = options
      .map(term => term.options?.[0]?.text || term.text)
      .join(' ');

    return corrected !== text ? corrected : null;
  }

  /**
   * More like this - related products
   */
  async moreLikeThis(productId, limit = 10) {
    const result = await this.client.search({
      index: INDEX_ALIAS,
      body: {
        size: limit,
        query: {
          more_like_this: {
            fields: ['name', 'description', 'tags', 'category.name'],
            like: [{ _index: INDEX_ALIAS, _id: productId }],
            min_term_freq: 1,
            max_query_terms: 12,
            min_doc_freq: 2,
            boost_terms: 1
          }
        },
        filter: [
          { term: { status: 'active' } },
          { must_not: { ids: { values: [productId] } } }
        ]
      }
    });

    return result.hits.hits.map(hit => ({
      id: hit._id,
      score: hit._score,
      ...hit._source
    }));
  }
}

module.exports = new SearchService();
```

---

## 7. Bulk Indexing

```javascript
// src/services/bulk.indexer.js

const esClient = require('../client/elasticsearch.client');

const INDEX_ALIAS = 'products_alias';
const BATCH_SIZE = 1000;

class BulkIndexer {
  constructor() {
    this.client = null;
    this.successCount = 0;
    this.errorCount = 0;
    this.errors = [];
  }

  initialize() {
    this.client = esClient.getClient();
  }

  /**
   * Bulk index documents
   */
  async bulkIndex(documents) {
    const operations = documents.flatMap(doc => [
      { index: { _index: INDEX_ALIAS, _id: doc.id } },
      this._transformDocument(doc)
    ]);

    const result = await this.client.bulk({
      body: operations,
      refresh: 'wait_for'
    });

    if (result.errors) {
      result.items.forEach((item, idx) => {
        if (item.index?.error) {
          this.errorCount++;
          this.errors.push({
            documentId: documents[idx]?.id,
            error: item.index.error
          });
        } else {
          this.successCount++;
        }
      });
    } else {
      this.successCount += documents.length;
    }

    return {
      indexed: this.successCount,
      errors: this.errorCount,
      errorDetails: this.errors
    };
  }

  /**
   * Index large dataset in batches
   */
  async bulkIndexLarge(documentStream) {
    let batch = [];
    let totalProcessed = 0;

    for await (const doc of documentStream) {
      batch.push(doc);

      if (batch.length >= BATCH_SIZE) {
        await this.bulkIndex(batch);
        totalProcessed += batch.length;
        console.log(`Indexed ${totalProcessed} documents...`);
        batch = [];

        // Small delay เพื่อไม่ overwhelm cluster
        await new Promise(resolve => setTimeout(resolve, 100));
      }
    }

    // Index remaining
    if (batch.length > 0) {
      await this.bulkIndex(batch);
      totalProcessed += batch.length;
    }

    return {
      total: totalProcessed,
      indexed: this.successCount,
      errors: this.errorCount
    };
  }

  /**
   * Transform document ให้ตรงกับ mapping
   */
  _transformDocument(product) {
    return {
      id: product.id,
      sku: product.sku,
      name: product.name,
      description: product.description,
      shortDescription: product.short_description,
      price: parseFloat(product.price),
      originalPrice: product.original_price ? parseFloat(product.original_price) : null,
      discountPercent: product.discount_percent || 0,
      category: {
        id: product.category_id,
        name: product.category_name,
        path: product.category_path
      },
      brand: product.brand_id ? {
        id: product.brand_id,
        name: product.brand_name
      } : null,
      tags: product.tags || [],
      attributes: (product.attributes || []).map(attr => ({
        key: attr.key,
        value: attr.value,
        displayName: attr.display_name
      })),
      metrics: {
        viewCount: product.view_count || 0,
        salesCount: product.sales_count || 0,
        rating: product.average_rating || 0,
        reviewCount: product.review_count || 0,
        wishlistCount: product.wishlist_count || 0
      },
      inStock: product.stock_quantity > 0,
      stockQuantity: product.stock_quantity || 0,
      status: product.status,
      createdAt: product.created_at,
      updatedAt: product.updated_at,
      // Completion suggester data
      suggest: {
        input: [
          product.name,
          product.brand_name,
          ...(product.tags || [])
        ].filter(Boolean),
        weight: Math.min(100, product.sales_count || 1)
      }
    };
  }
}

module.exports = new BulkIndexer();
```

---

## 8. Sync จาก PostgreSQL → Elasticsearch

```javascript
// src/sync/postgres-to-elasticsearch.sync.js

const { Pool } = require('pg');
const BulkIndexer = require('../services/bulk.indexer');
const esClient = require('../client/elasticsearch.client');

class PostgresToElasticsearchSync {
  constructor() {
    this.pool = new Pool({
      connectionString: process.env.DATABASE_URL
    });
    this.lastSyncAt = null;
  }

  /**
   * Full sync - ทำ initial load หรือ full reindex
   */
  async fullSync() {
    console.log('Starting full sync from PostgreSQL...');
    const startTime = Date.now();

    const client = await this.pool.connect();

    try {
      // ใช้ cursor สำหรับ large datasets
      await client.query('BEGIN');

      const cursor = client.query(
        new (require('pg-cursor'))(this._getFullSyncQuery())
      );

      async function* cursorToGenerator(cursor, batchSize = 1000) {
        while (true) {
          const rows = await cursor.read(batchSize);
          if (rows.length === 0) break;
          yield* rows;
        }
        await cursor.close();
      }

      BulkIndexer.initialize();
      const result = await BulkIndexer.bulkIndexLarge(
        cursorToGenerator(cursor)
      );

      await client.query('COMMIT');

      this.lastSyncAt = new Date();
      const duration = (Date.now() - startTime) / 1000;

      console.log(`Full sync completed in ${duration}s:`, result);
      return result;

    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  /**
   * Incremental sync - sync เฉพาะ records ที่ updated หลัง lastSyncAt
   */
  async incrementalSync() {
    if (!this.lastSyncAt) {
      return this.fullSync();
    }

    console.log(`Incremental sync since ${this.lastSyncAt}`);

    const { rows } = await this.pool.query(
      this._getIncrementalSyncQuery(),
      [this.lastSyncAt]
    );

    if (rows.length === 0) {
      console.log('No changes to sync');
      return { total: 0, indexed: 0 };
    }

    // Handle deletes
    const deletedProducts = rows.filter(r => r.deleted_at !== null);
    const updatedProducts = rows.filter(r => r.deleted_at === null);

    // Bulk delete
    if (deletedProducts.length > 0) {
      await this._bulkDelete(deletedProducts.map(p => p.id));
    }

    // Bulk upsert
    if (updatedProducts.length > 0) {
      BulkIndexer.initialize();
      await BulkIndexer.bulkIndex(updatedProducts);
    }

    this.lastSyncAt = new Date();

    return {
      total: rows.length,
      updated: updatedProducts.length,
      deleted: deletedProducts.length
    };
  }

  async _bulkDelete(ids) {
    const client = esClient.getClient();
    const operations = ids.flatMap(id => [
      { delete: { _index: 'products_alias', _id: id } }
    ]);

    await client.bulk({ body: operations });
    console.log(`Deleted ${ids.length} documents from Elasticsearch`);
  }

  _getFullSyncQuery() {
    return `
      SELECT
        p.id,
        p.sku,
        p.name,
        p.description,
        p.short_description,
        p.price,
        p.original_price,
        p.discount_percent,
        p.status,
        p.stock_quantity,
        p.view_count,
        p.sales_count,
        p.wishlist_count,
        p.created_at,
        p.updated_at,
        p.deleted_at,
        c.id as category_id,
        c.name as category_name,
        c.path as category_path,
        b.id as brand_id,
        b.name as brand_name,
        COALESCE(r.average_rating, 0) as average_rating,
        COALESCE(r.review_count, 0) as review_count,
        ARRAY_AGG(DISTINCT t.name) FILTER (WHERE t.name IS NOT NULL) as tags,
        JSON_AGG(
          DISTINCT jsonb_build_object(
            'key', pa.attribute_key,
            'value', pa.attribute_value,
            'display_name', pa.display_name
          )
        ) FILTER (WHERE pa.id IS NOT NULL) as attributes
      FROM products p
      LEFT JOIN categories c ON p.category_id = c.id
      LEFT JOIN brands b ON p.brand_id = b.id
      LEFT JOIN product_ratings r ON p.id = r.product_id
      LEFT JOIN product_tags pt ON p.id = pt.product_id
      LEFT JOIN tags t ON pt.tag_id = t.id
      LEFT JOIN product_attributes pa ON p.id = pa.product_id
      WHERE p.deleted_at IS NULL
      GROUP BY p.id, c.id, c.name, c.path, b.id, b.name, r.average_rating, r.review_count
      ORDER BY p.id
    `;
  }

  _getIncrementalSyncQuery() {
    return `
      SELECT
        p.id,
        p.sku,
        p.name,
        p.description,
        p.short_description,
        p.price,
        p.original_price,
        p.discount_percent,
        p.status,
        p.stock_quantity,
        p.view_count,
        p.sales_count,
        p.wishlist_count,
        p.created_at,
        p.updated_at,
        p.deleted_at,
        c.id as category_id,
        c.name as category_name,
        c.path as category_path,
        b.id as brand_id,
        b.name as brand_name,
        COALESCE(r.average_rating, 0) as average_rating,
        COALESCE(r.review_count, 0) as review_count,
        ARRAY_AGG(DISTINCT t.name) FILTER (WHERE t.name IS NOT NULL) as tags,
        JSON_AGG(
          DISTINCT jsonb_build_object(
            'key', pa.attribute_key,
            'value', pa.attribute_value,
            'display_name', pa.display_name
          )
        ) FILTER (WHERE pa.id IS NOT NULL) as attributes
      FROM products p
      LEFT JOIN categories c ON p.category_id = c.id
      LEFT JOIN brands b ON p.brand_id = b.id
      LEFT JOIN product_ratings r ON p.id = r.product_id
      LEFT JOIN product_tags pt ON p.id = pt.product_id
      LEFT JOIN tags t ON pt.tag_id = t.id
      LEFT JOIN product_attributes pa ON p.id = pa.product_id
      WHERE p.updated_at > $1
      GROUP BY p.id, c.id, c.name, c.path, b.id, b.name, r.average_rating, r.review_count
      ORDER BY p.updated_at DESC
    `;
  }
}

module.exports = new PostgresToElasticsearchSync();
```

---

## 9. Search API Controller

```javascript
// src/controllers/search.controller.js

const SearchService = require('../services/search.service');
const { body, query, validationResult } = require('express-validator');

class SearchController {
  async search(req, res) {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }

    try {
      const {
        q: queryText,
        page = 1,
        limit = 20,
        sort = 'relevance',
        category,
        brand,
        price_min,
        price_max,
        rating_min,
        in_stock,
        ...attributeFilters
      } = req.query;

      // Parse brand IDs
      const brandIds = brand ? brand.split(',') : undefined;

      // Parse attribute filters (format: attr_สี=แดง,น้ำเงิน)
      const attributes = {};
      for (const [key, value] of Object.entries(attributeFilters)) {
        if (key.startsWith('attr_')) {
          const attrKey = key.replace('attr_', '');
          attributes[attrKey] = value.split(',');
        }
      }

      const result = await SearchService.search({
        query: queryText,
        page: parseInt(page),
        limit: Math.min(100, parseInt(limit)),
        sort,
        filters: {
          categoryId: category,
          brandIds,
          priceMin: price_min ? parseFloat(price_min) : undefined,
          priceMax: price_max ? parseFloat(price_max) : undefined,
          ratingMin: rating_min ? parseFloat(rating_min) : undefined,
          inStock: in_stock === 'true' ? true : undefined,
          attributes: Object.keys(attributes).length > 0 ? attributes : undefined
        }
      });

      // Log search query สำหรับ analytics
      this._logSearch(queryText, result.total, req);

      res.json({
        success: true,
        data: result
      });

    } catch (error) {
      console.error('Search error:', error);
      res.status(500).json({
        success: false,
        message: 'Search failed',
        error: process.env.NODE_ENV === 'development' ? error.message : undefined
      });
    }
  }

  async suggest(req, res) {
    const { q: text, limit = 10 } = req.query;

    if (!text || text.length < 2) {
      return res.json({ success: true, data: [] });
    }

    try {
      const suggestions = await SearchService.suggest(text, parseInt(limit));

      // Check "did you mean"
      const didYouMean = await SearchService.didYouMean(text);

      res.json({
        success: true,
        data: {
          suggestions,
          didYouMean
        }
      });
    } catch (error) {
      console.error('Suggest error:', error);
      res.status(500).json({ success: false, message: 'Suggest failed' });
    }
  }

  async moreLikeThis(req, res) {
    const { productId } = req.params;
    const { limit = 10 } = req.query;

    try {
      const related = await SearchService.moreLikeThis(productId, parseInt(limit));
      res.json({ success: true, data: related });
    } catch (error) {
      console.error('More like this error:', error);
      res.status(500).json({ success: false, message: 'Failed to get related products' });
    }
  }

  _logSearch(query, resultCount, req) {
    // Log สำหรับ search analytics (async, non-blocking)
    setImmediate(() => {
      console.log(JSON.stringify({
        type: 'search',
        query,
        resultCount,
        userId: req.user?.id,
        sessionId: req.sessionID,
        timestamp: new Date().toISOString()
      }));
    });
  }
}

module.exports = new SearchController();
```

---

## 10. Express Router

```javascript
// src/routes/search.routes.js

const express = require('express');
const router = express.Router();
const SearchController = require('../controllers/search.controller');
const { query } = require('express-validator');

const searchValidation = [
  query('q').optional().isString().trim().isLength({ max: 200 }),
  query('page').optional().isInt({ min: 1, max: 1000 }),
  query('limit').optional().isInt({ min: 1, max: 100 }),
  query('sort').optional().isIn(['relevance', 'price_asc', 'price_desc', 'newest', 'rating', 'popular']),
  query('price_min').optional().isFloat({ min: 0 }),
  query('price_max').optional().isFloat({ min: 0 })
];

router.get('/search', searchValidation, SearchController.search.bind(SearchController));
router.get('/suggest', SearchController.suggest.bind(SearchController));
router.get('/products/:productId/related', SearchController.moreLikeThis.bind(SearchController));

module.exports = router;
```

---

## 11. Package.json

```json
{
  "name": "search-service",
  "version": "1.0.0",
  "scripts": {
    "start": "node src/index.js",
    "dev": "nodemon src/index.js",
    "setup": "node src/scripts/setup-index.js",
    "sync": "node src/scripts/full-sync.js",
    "test": "jest"
  },
  "dependencies": {
    "@elastic/elasticsearch": "^8.11.0",
    "express": "^4.18.2",
    "express-validator": "^7.0.1",
    "pg": "^8.11.3",
    "pg-cursor": "^2.10.3",
    "dotenv": "^16.3.1"
  },
  "devDependencies": {
    "jest": "^29.7.0",
    "nodemon": "^3.0.2"
  }
}
```

---

## สรุป

| หัวข้อ | รายละเอียด |
|--------|-----------|
| **Elasticsearch version** | 8.11.0 พร้อม ICU plugin สำหรับ Thai |
| **Index Strategy** | Alias-based สำหรับ zero-downtime reindex |
| **Thai Language** | Custom analyzer: thai tokenizer + stopwords + synonyms |
| **Multi-field Search** | name^4, brand^3, category^2, description^1 |
| **Boosting** | function_score ด้วย salesCount + rating + recency |
| **Faceted Search** | Aggregations: categories, brands, price ranges, attributes |
| **Autocomplete** | Completion suggester + edge ngram + prefix query |
| **Bulk Indexing** | Batches of 1000 + cursor-based streaming |
| **Sync Strategy** | Full sync (initial) + Incremental sync (scheduled) |
| **ILM** | Hot→Warm→Cold→Delete สำหรับ log indices |

**Next: Part 28 - API Design Best Practices & Versioning**
