ma:wizard
=========

ma:wizard is a smart package for [Meteor](https://www.meteor.com/) which simplifies data entry procedures in your app. It provides reactive data validation and lets you manage precisely the workflow of operations, giving you total control over the UI while providing little modular components that automatically integrate the package functionalities.

ma:wizard makes use of [maSimpleSchema](https://github.com/aldeed/meteor-simple-schema) for schemas definition and validation both client side and server side.

## Installation
Clone the repository in a local directory
````bash
$ cd ~/my_repos/
$ git clone https://github.com/doubleslashG/ma-wizard.git
````

Copy the directory in the `packages` folder of your Meteor project or make a symlink (better alternative to simplify the update procedure)
````bash
$ ln -s ~/my_repos/ma:wizard /my_project_path/packages/ma:wizard
````

Finally run
````bash
$ meteor add ma:wizard
````

Enjoy! :)

## Basics
Once the package is installed, the global `maWizard` object is available. This provides methods to work with the UI and to access inserted data, which are saved to the 'data context' of `maWizard`, a reactive data source which reproduces the database record but that is held in memory and can be accessed reactively by calling `maWizard.getDataContext()`.
`maWizard` should be initialized before use by calling the `init(conf)` method on it, which takes as parameter a configuration object with the following fields:

* `collection`: the MongoDB collection to work with.
* `id`: Optional. `_id` of the document to load from database, if any.
* `schema`: Optional. maSimpleSchema object related to `collection`. If not specified, `maWizard` expects to find the schema definition attached to the collection itself (through the maSimpleSchema method `attachSchema`).
* `template`: Optional. If you use the UI components provided by `maWizard`, you should specify the name of the template that is using them.
* `baseRoute`: Optional. Set a base route referred to by standard actions.

### Create mode
If no `id` is specified in `conf`, `maWizard` is initialized in "create" mode.  This means that the data context is initialized with an object built from the schema and whose values are default. The `_id` field will be `undefined` until the `maWizard.create()` method is called.
Example:
````javascript
maWizard.init({
	collection: Country,
	template: "countryView"
}
````
Calling the `create()` method a new document is inserted into the database and the corresponding `_id` is set in the data context, thus switching to "update" mode.


### Update mode
If an `id` is specified in `conf` or after calling `create()`, the right document is read from database and saved to the data context. You can then update the data as wanted without affecting the database. To make the final changes persistent, call `maWizard.saveToDatabase()`.

## Standard components
Standard components are handy ready-to-use templates that work out of the box with `maWizard`. As an example, consider the 'country' schema:
````javascript
CountrySchema = new maSimpleSchema({
	name: {
		type: String,
		label: "Name",
		max: 200,
		maDependencies: ["majorAirports"]
	},
	majorAirports: {
		type: [String],
		label: "Major airports"
	}
});
````
To let the user add a country just setting the name, we must initialize `maWizard` in "create" mode:
````javascript
maWizard.init({
	collection: Countries,
	schema: CountrySchema,
	template: "countryView"
});
````
Then, for the UI, we use a `maWizardTextInput`:
````HTML
<template name="countryView">
	{{> maWizardTextInput field="name" label="Country name" placeholder="Set country name..."}}
	{{> maWizardCreate}}
</template>
````
This is enough to provide the user with a text input to insert the country name (the `maWizardTextInput` template) and a button to insert the new entry into the database (the `maWizardCreate` template).
Using standard components, inserted values are automatically validated and saved to the data context on the `change` event (which fires when the user changes the value of the component and then focus is lost). If a value is invalid, an error message is shown and the component is styled with the `has-error` Bootstrap3 class.

### maWizardTextInput
A simple text input. It accepts both characters and numbers, though the validation is coherent with the data type reported in the schema definition.

### maWizardTextarea
A simple textarea. It accepts both characters and numbers, though the validation is coherent with the data type reported in the schema definition.

### maWizardCheckbox
A simple checkbox to deal with boolean values. This template just accepts the `field` and `label` parameters.

### maWizardSelect
A `<select>` element whose options are specified by the `values` parameter:
````HTML
{{> maWizardSelect field="genre" label="Genre" placeholder="Choose a genre" values=arrayOfGenres }}
````
If the `values` parameter is not specified, the options are read from the `maAllowedValues` or `allowedValues` field in the schema definition (in the reported order) by using the `maWizard.getSimpleSchemaAllowedValues(field)` method.

#### maWizardMultiselect
This component works the same as `maWizardSelect`, but lets you select more then one option. Works with schema entries whose `type` is an array (as `[String]`).

### maWizardCreate
Button to save the current data context in the database creating a new document. If you use this in "update" mode, you could duplicate database entries. Usually, you want to conditionally display this element checking for the `_id` field in the data context (see example for `maWizardSave`).

### maWizardSave
Button to update an existing document in the database with the values present in the data context. Usually, you want to conditionally display this element if we are in "update" mode:
````javascript
{#if getDataContextFieldHelper '_id'}}
	{{> maWizardCreate }}
{{else}}
	{{> maWizardSave }}
{{/if}}
````

### maWizardCancel & maWizardBackToList
Both templates display a button that redirects to the `baseRoute` through `iron:router`. If no `baseRoute` has been specified in `init(conf)`, redirects to homepage ("/").
The difference between the two templates is that they respectively report the labels "Cancel" and "Back to list" (guess the meaning of the second one).

### maWizardDelete
Button to delete the current document from database. To use in "update" mode.

## Attaching standard actions to other components
If you want to define your own components for the standard Create, Save, Cancel, BackToList and Delete actions, just add the boolean attribute `data-ma-wizard-actionName` to the chosen component, where `actionName` is one among `create`, `save`, `cancel`, `backToList` and `delete`.
As an example, here is the definition of the `maWizardSave` template:
````HTML
<template name="maWizardSave">
  <button class="btn btn-default" data-ma-wizard-save>Save</button>
</template>
````

## Custom components
If the standard components don't fit your needs, you can easily define fancy custom components that are automatically managed by `maWizard`. To let `maWizard` be aware of the existence of your custom component, just add the boolean attribute `data-ma-wizard-control` to it. Then to link the component to a certain schema field, use the attribute `data-schemafield="fieldName"`. The value stored in the data context by `maWizard` is read from the `value` attribute of the HTML component.

Sometimes you want a greater control over your custom components and you don't want `maWizard` to automatically manage them. In such a case, you can take complete control and use the `maWizard` APIs (see example later).

## maWizard Helpers
A set of reactive helpers to use with custom components.

### maWizardGetFieldValue(field)
Reactively gets the value of the specified schema field.

### maWizardFieldValidity(field)
If the field is invalid, returns `"has-error"` (this is the Bootstrap3 class attached to the standard components when the corresponding field value is invalid). Returns the empty string otherwise.
This is a reactive method that runs every time the specified field is validated by `maWizard`.

### maWizardErrMsg(field)
If the field is invalid, returns an appropriate error message (as specified in `maSimpleSchema`). Returns the empty string otherwise.
This is a reactive method that runs every time the specified field is validated by `maWizard`.

### maWizardAllowedValuesFromSchema(field)
Returns an array of label/value pairs read from the `maAllowedValues` or `allowedValues` field in the schema definition, with priority given to `maAllowedValues`.


