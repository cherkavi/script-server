<template>
    <div class="input-field" :title="config.description" :data-error="error">
        <select
                :id="config.name"
                ref="selectField"
                class="validate"
                :required="config.required"
                :multiple="config.multiselect"
                :disabled="options.length === 0">
            <option :selected="!anythingSelected" value="" disabled>Choose your option</option>
            <option v-for="option in options"
                    :value="option.value"
                    :selected="option.selected">{{ option.value }}</option>
        </select>
        <label :for="config.name">{{ config.name }}</label>
    </div>
</template>

<script>
    import * as M from 'materialize-css';
    import {addClass, contains, findNeighbour, isEmptyString, isNull, removeClass} from '../common';

    export default {
        props: {
            'config': Object,
            'value': [String, Array]
        },

        data: function () {
            return {
                options: [],
                anythingSelected: false,
                error: '',
                comboboxWrapper: null
            }
        },

        watch: {
            'config.values': {
                immediate: true,
                handler(newOptionValues) {
                    if (isNull(newOptionValues)) {
                        this.options = [];
                        return;
                    }

                    var newOptions = [];
                    for (var i = 0; i < newOptionValues.length; i++) {
                        newOptions.push({
                            value: newOptionValues[i],
                            selected: false
                        });
                    }

                    this.options = newOptions;

                    if (!this._fixValueByAllowedValues(this.config.values)) {
                        this._selectValue(this.value);
                    }

                    this.$nextTick(function () {
                        this.rebuildCombobox();
                    }.bind(this));
                }
            },

            'value': {
                immediate: true,
                handler(newValue) {
                    if (!this._fixValueByAllowedValues(this.config.values)) {
                        this._selectValue(newValue);
                    }
                }
            }
        },

        mounted: function () {
            // for some reason subscription in template (i.e. @change="..." doesn't work for select input)
            $(this.$refs.selectField).on('change', function () {
                var value;
                if (this.config.multiselect) {
                    value = $(this.$refs.selectField).val();
                } else {
                    value = this.$refs.selectField.value;
                }
                this.emitValueChange(value);
            }.bind(this));

            this.rebuildCombobox();

            this.$watch('error', function (errorValue) {
                var inputField = findNeighbour(this.$refs.selectField, 'input');
                if (!isNull(inputField)) {
                    inputField.setCustomValidity(errorValue);
                }
            }, {
                immediate: true
            })
        },

        beforeDestroy: function () {
            const instance = M.FormSelect.getInstance(this.$refs.selectField);
            instance.destroy();
        },

        methods: {
            emitValueChange(value) {
                this._validate(this.asArray(value));
                this.$emit('input', value);
            },

            _fixValueByAllowedValues(allowedValues) {
                if (isNull(this.value) || (this.value === '') || (this.value === [])) {
                    return false;
                }

                var newValue;
                if (this.config.multiselect) {
                    if (!Array.isArray(this.value)) {
                        if (contains(allowedValues, this.value)) {
                            return false;
                        }
                        newValue = [this.value];
                    } else {
                        newValue = [];
                        for (var i = 0; i < this.value.length; i++) {
                            var valueElement = this.value[i];
                            if (contains(allowedValues, valueElement)) {
                                newValue.push(valueElement)
                            }
                        }

                        if (newValue.length === this.value.length) {
                            return false;
                        }
                    }
                } else {
                    if (contains(allowedValues, this.value)) {
                        return false;
                    }

                    newValue = null;
                }

                this.emitValueChange(newValue);
                return true;
            },

            _selectValue(value) {
                var selectedValues = this.asArray(value);

                this.anythingSelected = false;

                var existingSelectedValues = new Set();

                for (var i = 0; i < this.options.length; i++) {
                    var option = this.options[i];
                    option.selected = contains(selectedValues, option.value);

                    if (option.selected) {
                        this.anythingSelected = true;
                        existingSelectedValues.add(option.value);
                    }
                }

                this._validate(selectedValues);

                this.$nextTick(function () {
                    this.updateComboboxValue();
                }.bind(this));
            },

            _validate(selectedValues) {
                if (this.config.required && (selectedValues.length === 0)) {
                    this.error = 'required';
                } else {
                    this.error = '';
                }
                this.$emit('error', this.error);
            },

            rebuildCombobox() {
                this.comboboxWrapper = M.FormSelect.init($(this.$refs.selectField),
                    {dropdownOptions: {constrainWidth: false}})[0];

                $(this.$refs.selectField).siblings('ul').children('li').each(function () {
                    var text = $(this).children('span:first-child').text();
                    if (text) {
                        this.title = text;
                    }
                });

                $(this.$refs.selectField).siblings('ul').children('li.disabled')
                    .each(function () {
                        $(this).attr('tabindex', -1)
                    });

                this.updateComboboxValue();
            },

            updateComboboxValue() {
                const inputField = findNeighbour(this.$refs.selectField, 'input');

                // setCustomValidity doesn't work since input is readonly
                if (this.error) {
                    addClass(inputField, 'invalid');
                } else {
                    removeClass(inputField, 'invalid');
                }

                if (!isNull(this.comboboxWrapper)) {
                    this.comboboxWrapper._setValueToInput();
                    this.comboboxWrapper._setSelectedStates();
                }
            },

            asArray(value) {
                var valuesArray = value;

                if (isEmptyString(valuesArray)) {
                    valuesArray = [];
                } else if (!Array.isArray(valuesArray)) {
                    valuesArray = [valuesArray];
                }
                return valuesArray;
            }
        }
    }
</script>


