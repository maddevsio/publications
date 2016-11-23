
Привет. Однажды в рассылке Golang Weekly мне попался проект [Bleve](http://blevesearch.com).
Это полнотекстовый поиск, который написан на Go. Проект интересный, и появилось бешеное желание получить с ним опыт работы.

Bleve может хранить данные в разных embedded БД. Например:

1. BoltDB
2. LevelDB
3. RocksDB
4. Goleveldb
5. forestdb
6. Gtreap

По умолчанию он использует BoltDB.

Работать с Bleve просто. Например:

```Go
import "github.com/blevesearch/bleve"

func main() {
    // Откроем новый индекс
    mapping := bleve.NewIndexMapping()
    index, err := bleve.New("example.bleve", mapping)

    // Положим не много данных
    err = index.Index(identifier, your_data)

    // Найдем что-нибудь
    query := bleve.NewMatchQuery("text")
    search := bleve.NewSearchRequest(query)
    searchResults, err := index.Search(search)
}
```

Все просто и понятно. Но выглядит оно не с реального мира.
Чтобы быть ближе к реальному миру, сделаем бота для [Slack](https://slack.com), который будет хранить историю.

## Архитектура бота

1. Сервис для работы со slack
2. Сервис индекс. Для хранения и поиска сообщений

## План
1. Берем Bleve для поиска и хранения истории
2. Берем  https://api.slack.com/methods/channels.history для работы со слаком
3. Если к нам пришло сообщение не по меншну бота - кладем в индекс
4. Если пришло сообщение с меншном - чистим и ищем по текущему каналу


## Slack

Со слаком все просто и пример по сути будет чуть сложнее, чем [пример из репо](https://github.com/nlopes/slack/blob/master/examples/websocket/websocket.go)

Единственное, что нам потребуется - два метода, чтобы проверить, адресовано ли боту сообщение и очистить его от имени бота

```Go
import "strings"
func isToMe(message string) bool {
	return strings.Contains(message, fmt.Sprintf("<@%s>", ss.me))
}

func cleanMessage(message string) string {
	return strings.Replace(message, fmt.Sprintf("<@%s> ", ss.me), "", -1)
}
```

## Bleve

Учитывая то, что я люблю использовать goleveldb как встраиваемую БД для своих проектов. В этом проекте решил использовать ее же.

Хранить в Bleve будем данные посложнее в виде
```Go
type IndexData struct {
	ID        string `json:"id"`
	Username  string `json:"username"`
	Message   string `json:"message"`
	Channel   string `json:"channel"`
	Timestamp string `json:"timestamp"`
}
```

Создадим индекс с Goleveldb в качестве БД.

```Go
import (
  "github.com/blevesearch/bleve"
  "github.com/blevesearch/bleve/index/store/goleveldb"
)

func createIndex() (bleve.Index, error) {
	indexName := "history.bleve"
	index, err := bleve.Open(indexName)
	if err == bleve.ErrorIndexPathDoesNotExist {
		mapping := buildMapping()
		kvStore := goleveldb.Name
		kvConfig := map[string]interface{}{
			"create_if_missing": true,
		}

		index, err = bleve.NewUsing(indexName, mapping, "upside_down", kvStore, kvConfig)
	}
	if err != nil {
		return err
	}
}
```
  и метод buildMapping, который создаст нам mapping для хранения
```Go
func (ss *SearchService) buildMapping() *bleve.IndexMapping {
	ruFieldMapping := bleve.NewTextFieldMapping()
	ruFieldMapping.Analyzer = ru.AnalyzerName

	eventMapping := bleve.NewDocumentMapping()
	eventMapping.AddFieldMappingsAt("message", ruFieldMapping)

	mapping := bleve.NewIndexMapping()
	mapping.DefaultMapping = eventMapping
	mapping.DefaultAnalyzer = ru.AnalyzerName
	return mapping
}
```

С поиском все чуть сложнее.
```Go
func (ss *SearchService) Search(query, channel string) (*bleve.SearchResult, error) {
	stringQuery := fmt.Sprintf("/.*%s.*/", query)
  // NewTermQuery создает Query для нахождения значений в индексе, которые строго совпадают с запросом
	ch := bleve.NewTermQuery(channel)
  // Создаем Query для совпадений фраз в индексе. Анализатор выбирается по полю. Ввод анализируется этим анализатором. Токенезированные выражения от анализа используются для посторения поисковой фразы. Результирующие документы должны совпадать с этой фразой.
	mq := bleve.NewMatchPhraseQuery(query)
  // Создаем Query для поиска значений в индексе по регулярному выражению
	rq := bleve.NewRegexpQuery(query)
  // Создаем Query для поиска документов, результаты которого удовлетворят поисковой строке.
	qsq := bleve.NewQueryStringQuery(stringQuery)
  // Создаем составную Query Результат должен удовлетворять хотя бы одной Query.
	q := bleve.NewDisjunctionQuery([]bleve.Query{ch, mq, rq, qsq})
	search := bleve.NewSearchRequest(q)
	search.Fields = []string{"username", "message", "channel", "timestamp"}
	return ss.index.Search(search)
}
```
Соединив все вместе, мы получим бота, который сохраняет историю и может искать по ней без тяжеловесной жавы на примерах ElasticSearch, Solr.

Полный код проекта доступен на [Github](https://github.com/maddevsio/slack_history_bot")
