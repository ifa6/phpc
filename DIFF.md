# Difference between PHP 5 and PHP 7

## Object

### Common
- get object from `zval`s
  - 5: zend_object_store_get_object(d1 TSRMLS_CC)
  - 7: from offset : `(php_object_struct *)((char*)(obj) - XtOffsetOf(php_object_struct, std))`


### Handlers
- extra handlers in 7
  - `handlers.offset = XtOffsetOf(php_object_struct, std)`
  - `handlers.free_obj = object_free_storage_function

#### create_object_ex (ex method for clone and new)
- return
  - 5: `zend_object_value  (retval)`
  - 7: `zend_object *      (&intern->std)`
- intern
  - 5: `calloc(1, sizeof(php_object_struct))`
  - 7: `ecalloc(1, sizeof(php_object_struct) + sizeof(zval) * (class_type->default_properties_count - 1))`
- init props
  - 5: always
  - 7: only for new object (not for clone)
- save new intern
  - 5: for clone
  - 7: never (it's taken from offset)
- handlers
  - 5: `retval.handlers`
  - 7: `intern->std.handlers`
- object store
  - 5: `zend_objects_store_put(intern, (zend_objects_store_dtor_t)zend_objects_destroy_object, (zend_objects_free_object_storage_t) object_free_storage_function, NULL TSRMLS_CC)`
  - 7: none (it's set in init)

#### create_object
- return
  - 5: `zend_object_value  (retval)`
  - 7: `zend_object *      (&intern->std)`
- 2nd argument
  - 5: NULL (intern reference)
  - 7: 1    (whether to init props)

#### clone_obj
- return
  - 5: `zend_object_value  (retval)`
  - 7: `zend_object *      (&intern->std)`
- new object : `new_obj`
  - 5: reference arg to create_object_ex
  - 7: XtOffsetOf of the create_object_ex result
- new object value : `new_ov`
  - 5: result of create_object_ex
  - 7: none
- clone members
  - 5: `zend_objects_clone_members(&new_obj->std, new_ov, &old_obj->std, Z_OBJ_HANDLE_P(this_ptr) TSRMLS_CC)`
  - 7: `zend_objects_clone_members(&new_obj->std, &old_obj->std TSRMLS_CC)`

#### compare_object
- get object from `zval`s (see above)

#### get_gc
- table pointer (second argument)
  - 5: `zval ***table`
  - 7: `zval **table`

#### get_properties
- modifying existing props from `zend_std_get_properties`
  - 5: `zval *`: use `MAKE_STD_ZVAL` and then save
  - 7: `zval`: save as it is

#### free_storage
- get object from `zval`s (see above)
- free object
  - 5: `efree(intern)`
  - 7: do nothing

#### get_property_ptr_ptr
- return
  - 5: `zval **`: ptr of ptr to the engine zval or NULL if overloaded
  - 7: `zval *`: ptr to engine zval or NULL if overloaded
- 4th parameter
  - 5: `const zend_literal *key`
  - 7: `void **cache_slot`
- member name handling (2nd param)
  - 5: convert it to string zval if needed and free it if conversion changed zval
  - 7: get zend_string from zval and release the string at the end

#### read_property
- return
  - 5: `zval **`: ptr of ptr to the engine zval or NULL if overloaded
  - 7: `zval *`: ptr to engine zval or NULL if overloaded
- 4th parameter
	- 5: `const zend_literal *key`
	- 7: `void **cache_slot`
- extra `zval *rv` last (5th) parameter in 7
  - the same as zend_read_property
  - also when calling std handler
- member name handling (2nd param)
  - 5: convert it to string zval if needed and free it if conversion changed zval
  - 7: get zend_string from zval and release the string at the end
- we need to ensure in 5 that retval is temporary variable
  - `Z_SET_REFCOUNT_P(retval, 0); Z_UNSET_ISREF_P(retval);`


### Methods
- all have the common issue for getting object from zval (see above)

### __construct
- 7: It is possible to call `ZEND_CTOR_MAKE_NULL()` if init fails
