# Integration von zwei Micro-Frontends über Custom Elements am konkreten Beispiel
Im Folgenden wollen wir zeigen, wie wir Custom Elements zur Integration von Micro-Frontends in einem großen Projekt in der Finanzindustrie mit über 40 Self-Contained-Systems eingesetzt haben. 
Das zeigen wir exemplarisch an einer kleinen Beispiel-Anwendung, die aus zwei Self-Contained-Systems besteht.

## Die Beispiel-Anwendung "StockPrice": Korrelation von Aktienkursen
Mit der aus zwei Micro-Frontends bestehenden Beispiel-Anwendung "StockPrice" kann man sich Charts erzeugen lassen, die die Aktienkurse (Jahre 2014 bis 2018) von zwei Firmen als Scatterplot inklusive Korrelationskoeffizienten darstellen:  

<img src="https://cdn.jsdelivr.net/gh/owidder/jsArtikel@ow20190515-01/oliver/correlationApp.png"/>

Dazu kann man in den beiden Automplete-Eingabefeldern am oberen Rand jeweils eine Firma (aus dem Dow Jones) eingeben. Hat man beide Firmen eingegeben, wird der Chart angezeigt. 
Live kann man die Anwendung hier sehen: [http://bit.ly/stockprice-page](http://bit.ly/stockprice-page)

## Die Self-Contained-Systems "Company" und "StockHistory"
Die Anwendung "StockPrice" besteht aus den zwei Self-Contained-Systems (SCS) "Company" und "StockHistory", deren Micro-Frontends auf einer Web-Seite mit Hilfe von Custom Elements integriert werden:
1. SCS "Company": Stellt einen Service zur Verfügung, über den Namen und Abkürzungen aller Dow-Jones-Companies abgeholt werden können. Das Micro-Frontend ist ein Eingabefeld mit Autocompletion.
	Es kann über ein Custom Element mit Namen `<select-company/>` eingebunden werden kann.
2. SCS "StockHistory": Stellt einen Service zur Verfügung, über den die historischen Aktienkurse abgefragt werden können. Das Micro-Frontend ist der Scatterplot-Chart. 
	Er kann über ein Custom Element mit Namen `<company-correlation/>` eingebunden werden.

Jedes Self-Contained-System liefert sein Micro-Frontend (JavaScript-File mit dem Code des Custom Elements) selber aus. 
So hostet z.B. das System "Company" das JavaScript-File `selectComponentElement.js`, das den Code für das Custom Element `<select-company/>` enthält.
Jedes Custom Element greift wiederum nur auf Services des Self-Contained-Systems zu, von dem es ausgeliefert wurde.

<img src="https://cdn.jsdelivr.net/gh/owidder/jsArtikel@ow20190520-01/oliver/StockPrice.png"/>

* In diesem Beispiel nutzt das System "StockPrice" die Micro-Frontends von "Company" und "StockHistory". Genauso könnte aber auch "StockHistory" ein Micro-Frontend von "Company" nutzen oder umgekehrt.
* Wir gehen hier davon aus, dass alle Self-Contained-Systems hinter einem Reverse-Proxy unterhalb derselben Domain zu erreichen sind. Das ist für Anwendungen, die aus vielen Self-Contained-Systems bestehen nicht ungewöhnlich. 
	(So schließt man auch mögliche Probleme mit CORS-Einschränkungen des Browsers aus) 

## Das Micro-Frontend von "Company"
Das SCS "Company" stellt den Service "companies" zur Verfügung. Er liefert Namen und Abkürzungen aller Firmen aus dem Dow Jones:
```
[
   {"short":"MMM","full":"3M Company"},
   {"short":"AOS","full":"A.O. Smith Corp"},
   ...
]
```

Neben dem Service "companies" liefert das SCS "Company" auch ein Micro-Frontend aus. Dieses Micro-Frontend (ein JavaScript-File) enthält ein Custom Element mit Namen `<select-company/>`, das wiederum den Service "companies" aufruft. 

<img src="https://cdn.jsdelivr.net/gh/owidder/jsArtikel@ow20190520-02/oliver/company.png"/>

`<select-company/>` rendert sich als Eingabefeld mit Autocompletion-Funktionalität, über das eine Firma aus dem Dow Jones eingegeben werden kann.

<img src="https://cdn.jsdelivr.net/gh/owidder/jsArtikel@ow20190516-02/oliver/selectCompany.png"/>

Verwenden kann man das Element z.B. so:
```
<select-company></select-company>
<div id="company"></div>
<script>
var element = document.querySelector("select-company");
element.onChangeCompany = function (company) {
	var companyDiv = document.querySelector("div#company");
	companyDiv.innerHTML = "Company = " + company.full + 
		" [" + company.short + "]";
}
</script>
```
Dem Custom Element `<select-company/>` kann als Property `onChangeCompany` eine Callback-Function übergeben werden. Diese wird nach der Eingabe durch den Anwender mit dem entsprechenden Company-Object (z.B.: `{short: "AAPL", full: "Apple Inc."}`) aufgerufen.

Live kann man sich das Element hier ansehen: [http://bit.ly/stockprice-company](http://bit.ly/stockprice-company)

## Das Custom Element "select-company"
Bei der Implementierung der Custom Elements haben wir die folgenden Prinzipien umgesetzt:
* Kein Shadow-DOM: Wir haben bewusst auf den Shadow-DOM verzichtet. Dies hatte zwei Gründe:
	* Man soll der integrierten Anwendung nicht ansehen, dass sie aus vielen Micro-Frontends zusammengesetzt ist. Dies erfordert, dass für alle Custom Elements die selben CSS-Regeln gelten. Darum haben wir die kapselnde Eigenschaft des Shadow-DOM nicht benötigt.
	* Unverträglich des von uns eingesetzten UI-Frameworks *React* mit dem Shadow-DOM: Der Einsatz von React-Componenten innerhalb des Shadow-DOM führte zu Problemen bei der Event-Verarbeitung (siehe [http://bit.ly/react-shadow-dom](http://bit.ly/react-shadow-dom)).
* Custom Elements sind nur schmale Wrapper: Die gesamte client-seitige Funktionalität eines Micro-Frontends befindet sich innerhalb einer React-Component (mit ihren Unter-Compenents). 
	Ein Custom Elements ist immer nur ein schmaler Wrapper um diese React-Component. Dieses Prinzip hat es uns erlaubt, die Funktionalität der React-Components auch noch an anderen Stellen zu verwenden.
* Ein Custom Element ruft nur Services von der Adresse auf, von der es ausgeliefert wurde. 

Mit diesen Prinzipien sieht die Implementierung (`SelectCompanyElement.tsx`) des Custom Elements `<select-company/>` folgendermaßen aus:
```
import * as React from "react";  
import * as ReactDOM from "react-dom";  
  
import {SelectCompany, Company} from "./SelectCompany";  
  
const scriptPath = document.currentScript.getAttribute("src");
const parts = scriptPath.split("/");  
const basedir = parts.slice(0, parts.length-1).join("/");
  
class SelectCompanyElement extends HTMLElement {  
  
    public onChangeCompany: (Company) => void;    
  
    connectedCallback() {  
        ReactDOM.render(<SelectCompany 
	        basedir={basedir} 
	        onChange={(company: Company) => 
		        {this.onChangeCompany(company)}}/>, this);  
    }    
}  
customElements.define("select-company", SelectCompanyElement);
```
* `SelectCompany` ist die React-Component, in der sich die eigentliche Funktionalität des Micro-Frontends befindet
* `Company` ist ein einfaches Interface, das die Daten beschreibt, wie sie vom Service "companies" kommen:
```
export interface Company {  
    short: string;  
    full: string;  
}
```
* Gemäß dem Prinzip, dass alle Funktionalität in der Rect-Component liegt, findet auch der Aufruf des Service "companies" innerhalb der React-Component statt. Darum geben wir der React-Component über die Property `basedir` die Adresse mit, von der aus der Code des Custom Elements ausgeliefert wurde. Denn dort kann die React-Component auch den Service finden. 
	Diese Adresse holen wir uns über: 
	```
	const scriptPath = document.currentScript.getAttribute("src")
	```
	und schnibbeln dann einfach das letzte Pfad-Element (den Namen des JavaScript-Files) weg:
	```
	const parts = scriptPath.split("/");  
	const basedir = parts.slice(0, parts.length-1).join("/");
	```
* Wie oben beschrieben verwenden wir keinen Shadow-DOM, sondern rendern die DOM-Elemente des Custom Elements direkt in den den Main-DOM. Das sollte man aber nicht im Constructor des Custom Elements tun sondern in der `connectedCallback`-Lifecycle-Methode. 
	Dort rendern wir die React-Component:
	```
    connectedCallback() {  
        ReactDOM.render(<SelectCompany 
	        basedir={basedir} 
	        onChange={(company: Company) => 
		        {this.onChangeCompany(company)}}/>, this);  
    }    
	```
	Als Container-DOM-Element (zweiter Parameter der `ReactDOM.render()`-Methode) übergeben wir einfach `this`. Innerhalb der `connectedCallback`-Methode repräsentiert `this` das Root-Element des Custom Elements.
	
	Als `onChange`-Property übergeben wir eine Lambda-Function, die wiederum die Callback-Function aufruft, die dem Custom Element über die `onChangeCompany`-Property übergeben wurde.

## Die React-Component "SelectCompany"

Die React-Component `SelectCompany` enthält die eigentliche Funktionalität des Micro-Frontends:

```
import * as React from "react";  
import * as _ from "lodash";  
import {AutoComplete} from "antd";  
  
export interface Company {  
    short: string;  
    full: string;  
}  
  
interface Props {  
    initialShort?: string;  
    onChange: (company: Company)  => void;  
    basedir: string;  
}  
  
interface State {  
    companies: Company[];  
    data: string[];  
    value?: string;  
}  
  
export class SelectCompany extends React.Component<Props, State> {  
  
    readonly state: State = {data: [], companies: []};  
  
    handleSearch(value: string) {  
        const data = _.uniq(this.state.companies.map(
	        company => company.full).filter(full => 
		        full.toLowerCase().indexOf(value.toLowerCase()) > -1));  
        this.setState({data, value})  
    }  
  
    handleSelect(full: string) {  
        const selectedCompany = this.state.companies.find(
	        s => (s.full == full));  
        this.props.onChange(selectedCompany);  
        this.setState({value: full})  
    }  
  
    async componentDidMount() {  
        const companies = await fetch(
		        `${this.props.basedir}/../../service/companies`
	        ).then(resp => resp.json());  
        this.setState({companies});  
    }  
  
    render() {  
        return <div className="input-field">  
            <AutoComplete  
	            dataSource={this.state.data} 
	            onSearch={(text) => this.handleSearch(text)} 
	            onSelect={(value: string) => this.handleSelect(value)}
	            value={this.state.value}
	            placeholder="Enter company"/>  
        </div>  
    }  
}
```

* Als Eingabefeld verwenden wir `AutoComplete` aus der Bibliothek *Ant Design* (siehe [https://ant.design/](https://ant.design/))
* Nachdem sich die React-Component zum ersten Mal gerendert hat, wird in der React-Lifecycle-Methode `componentDidMount()` der Service "companies" aufgerufen. Das vom Service zurück gelieferte Array mit Namen und Abkürzungen des Companies wird in den State der React-Component gelegt, so dass das Eingabefeld erneut gerendert wird.
* In `handleSearch()` werden aus den vom Service geladenen Company-Namen, diejenigen gefiltert, die dem eingegebene Teilstring entsprechen, so dass `AutoComplete` eine Vorschlagsliste anzeigen kann.
* `handleSelect()` wird aufgerufen, wenn eine Company ausgewählt worden ist. Hier wird die vom Custom Element über die Property `onChange` übergebene Callback-Function  aufgerufen.

## Webpack

Mit folgender Webpack-Konfiguration lassen sich nun Custom Element und React-Component in eine JavaScript-File mit Namen `selectCompanyElement.js` packen, so dass es von der integrierenden Anwendung (hier "StockPrice") verwendet werden kann:

```
module.exports = {  
    mode: process.env.NODE_ENV,  
    entry: {  
        selectCompanyElement: "./src/SelectCompanyElement.tsx"  
  },  
    output: {  
        filename: "build/[name].js",  
    },  
    devtool: "source-map",  
    module: {  
        rules: [{test: /\.(ts|tsx)$/, use: "ts-loader"}]  
    },  
    resolve: {  
        extensions: [".tsx", ".ts", ".js"]  
    }  
}
```
Nach dem Aufruf von Webpack liegt im Verzeichnis `build` ein File mit Namen `selectCompanyElement.js`.

## Das Custom Element "company-correlation"

Die über die zwei Custom Elements `<select-company/>` ausgewählten Companies, können nun dem Custom Element `<company-correlation/>` - das Micro-Frontend des Self-Contained-Systems "StockHistory" - übergeben werden. 
Dazu hat `<company-correlation/>` die zwei Attribute `short-x` und `short-y`.
Über `short-x` kann man die Abkürzung der Company übergeben, die auf der X-Achse angezeigt werden soll. Analog übergibt man über `short-y` die Abkürzung der Firma, die auf der Y-Achse angezeigt werden soll. Vereinfacht sieht der Code der Gesamt-Anwendung "StockPrice" wie folgt aus:

```
<head>
<script src="./company/build/selectCompanyElement.js"></script>  
<script src="./stockHistory/build/companyCorrelationElement.js"></script>
</head>

<body>
<select-company id="selectCompany1"></select-company>
<select-company id="selectCompany2"></select-company>
<company-correlation id="companyCorrelation"></company-correlation>

<script>
document.getElementById("selectCompany1").onChangeCompany = 
	function(company) {
		document.getElementById("companyCorrelation")
			.setAttribute("short-y", company.short);
	}
document.getElementById("selectCompany2").onChangeCompany = 
	function(company) {
		document.getElementById("companyCorrelation")
			.setAttribute("short-x", company.short);
	}
</script>
</body>
```

Im Gegensatz zu `<select-company/>` besitzt `<company-correlation/>` Attribute, die sich auch mehrfach ändern können. Dafür müssen wir im Custom Element die Lifecycle-Methode `attributeChangedCallback()` implementieren:

```
class CompanyCorrelationElement extends HTMLElement {  
  
    static get observedAttributes() {  
        return ["short-x", "short-y"];  
    }  
  
    drawReactComponent() {  
        ReactDOM.render(<CompanyCorrelation
	        basedir={basedir}
		    shortX={this.getAttribute("short-x")}
		    shortY={this.getAttribute("short-y")}/>, this);  
    }  
  
    attributeChangedCallback() {  
        this.drawReactComponent();  
    }  
  
    connectedCallback() {  
        this.drawReactComponent();  
    }  
}  
  
customElements.define("company-correlation", 
	CompanyCorrelationElement);
```

* Über `static get observedAttributes()` teilen wir der Custom-Element-API mit, für welche Attribute wir uns interessieren und über Änderungen informiert werden wollen. Mit jeder Änderung eines Attributes wird dann `attributeChangedCallback()` aufgerufen.
* Im Sinne unseres Small-Wrapper-Principles tun wir in `attributeChangedCallback()` nichts weiter, als die React-Component mit den neuen Properties erneut zu rendern.

## Die React-Component "CompanyCorrelation"

Das Custom Element `<company-correlation/>` gibt die Attribute `short-x` und `short-y` an die React-Component als Properties `shortX` bzw. `shortY` weiter. Diese lädt bei jeder Änderung ihrer Properties (React-Lifecycle-Methode `componentDidUpdate()`) die historischen Kursdaten der jeweilige Company über einen Service-Call vom System "StockHistory":

<img src="https://cdn.jsdelivr.net/gh/owidder/jsArtikel@ow20190520-03/oliver/companyCorrelation.png"/>

Die Kursdaten werden dann im State der React-Component abgelegt:

```
export interface EndOfDayPrice {  
    date: string;  
    price: number;  
}

interface Props {  
    shortX: string;  
    shortY: string;  
    basedir: string;  
}  
  
interface State {  
    pricesX: EndOfDayPrice[];  
    pricesY: EndOfDayPrice[];  
}

export class CompanyCorrelation extends
	React.Component<Props, State> {  
  
    readonly state: State = {pricesX: [], pricesY: []}

	async loadData(symbol: string): Promise<EndOfDayPrice[]> {  
	    const response = await 
		    fetch(`${this.props.basedir}/../../service/${symbol}`);  
	    return await response.json();  
	}  
  
	async componentDidUpdate() {  
        const pricesX = await this.loadData(this.props.shortX);  
        const pricesY = await this.loadData(this.props.shortY);  
        this.setState({pricesX, pricesY});  
    }  
    ...

	render() {
		...
	}
}
```

Der komplette Code der React-Component `CompanyCorrelation` ist etwas komplexer. Der Einfachheit wegen soll er hier nicht im Einzelnen gezeigt werden. Interessierte finden den Code unter [http://bit.ly/companyCorrelation](http://bit.ly/companyCorrelation).

## Fazit

Unsere Erfahrungen mit Custom Elements für die Integration von Micro-Frontends waren weitestgehend positiv. Dennoch hat jede Münze (mindestens) zwei Seiten:

Vorteile:
* Komplett unabhängiges Deployment: Da die Custom Elements erst zur Laufzeit geladen werden, kann ein Self-Contained-System seine Micro-Frontends ändern und neu deployen, ohne dass eines der Systeme, die das Micro-Frontend einbinden, neu deployt werden muss. 
* Einfache Schnittstellen: Das nutzende System 'unterhält' sich mit dem Micro-Frontend in den meisten Fällen nur über die Properties des Custom Elements. (Falls das Micro-Frontend jedoch auch Informationen zurück liefern soll, wie z.B. bei `<select-company/>` benötigt man als komplexere Schnittstelle eine Callback-Function)

Nachteile:
* Seiten, die viele Micro-Frontends einbinden, müssen von vielen System JavaScript-Files laden
* Zusätzliche Komplexität durch Custom Elements
* Werden Micro-Frontends mehrfach eingebunden, werden ggf. mehrfach identischen Server-Calls ausgeführt
	* Z.B. führen die beiden Custom Elements `<select-company/>` auf der StockPrice-Page zweimal den gleichen Aufruf des Service "companies" aus. Dies kann man verhindern, was aber zu zusätzlicher Komplexität führt.
<!--stackedit_data:
eyJoaXN0b3J5IjpbMzExMTQyOTc0LDc5MjE2NTUyMCwxNDAwOT
MwNTU3LC0xMDg1NTYzMTg4LDk2OTEyNjY3NiwtMjEyOTA2Nzc3
LC01ODg4NjEyMjAsNDAzMDE2NDMxLC03MTE2OTU3OTYsOTQwOD
g2MzYwLC02Njk4OTU1MjgsMTA1MTI0Njc1OCwtNTUyMTA5NDEx
LDcwMzQzNjc1NSwtMjExNTUzODU3MSwxMjI3ODMyMDI4LDc1Nj
YzNzU1LDEzMjA0NjY1OTEsMTQ2MTI0NjUyNCwxMjQ1NjUwMjYw
XX0=
-->