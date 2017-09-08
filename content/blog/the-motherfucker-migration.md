+++
date = "2017-08-24T19:49:30+02:00"
title = "The motherfucker migration"
tags = [ "development", "django", "migrations", "python" ]
categories = [ "development" ]
series = [ "django" ]
draft = false

+++

This (my very first) post covers what I've come to refer as the *motherfucker* migration in Django. You can find the accompanying code, as well as detailed instructions on how to apply it, [here](https://github.com/salvalcantara/django-change-pk).

For the sake of explanation, let's consider a Django app with the following two (oversimplified) related models:

```python
class Author(models.Model):
    name = models.CharField(max_length=30, primary_key=True)
```

```
class Article(models.Model):
    title = models.CharField(max_length=140)
    authors = models.ManyToManyField(Author, related_name='articles')
```

In short, authors write articles, and an article can have one or more authors. Note the following crucial point: the Author model specifies *name* as a custom primary key (PK) field, instead of relying on Django for generating the default (autoincrement) PK as done in Article.[^1] So, since *name* acts as the PK for the authors table, one cannot change an author's name; intead, a new author would be created.[^2] This is clearly undesirable and you might be wondering why someone would want to mess up the database in the first place. Well, the reasons behind such a pitfall are beyond the scope of this post, but suffice it to say that I have come accross this problem myself, and I am not apparently alone.[^3] 

Now, what can we do about it? First thing would be to fix the Author class:

```python
class Author(models.Model):
    name = models.CharField(max_length=30, unique=True)
```

By replacing `primary_key=True` with `unique=True` we are implicitly using the default autoincrement `id` field as the new PK.[^1] This minor change alone won't do the trick, however, as confirmed by the output of the `manage.py makemigrations` command: 

```
You are trying to add a non-nullable field 'id' to author without a default; 
we can't do that (the database needs something to populate existing rows).
Please select a fix:
 1) Provide a one-off default now (will be set on all existing rows)
 2) Quit, and let me add a default in models.py 
```

To make things even worse, we also have the pivot table and the corresponding foreign key constraint. Addressing all these nuances requires us to write a custom (*motherfucker*) migration, since there is no way that Django can generate it for us automatically. Moreover, since it is necessary to execute some database-specific DDL statements to change the *schema*, in this post we will restrict ourselves to MySQL.

At this point, it might help to look at some sample data:

```
mysql> select * from app_author;
+---------------+
| name          |
+---------------+
| A. Visioli    |
| C. Pedret     |
| F. Padula     |
| R. Vilanova   |
| S. Alcántara  |
| S. Skogestad  |
+---------------+
```

```
mysql> select * from app_article;
+----+-------------------------------------------------------------------------------+
| id | title                                                                         |
+----+-------------------------------------------------------------------------------+
|  1 | PID control in terms of robustness/performance and servo/regulator trade-offs |
|  2 | H-infinity control of fractional linear systems                               |
|  3 | Simple analytic rules for model reduction and PID controller tuning           |
+----+-------------------------------------------------------------------------------+
```

```
mysql> select * from app_article_authors;
+----+------------+---------------+
| id | article_id | author_id     |
+----+------------+---------------+
|  1 |          1 | C. Pedret     |
|  3 |          1 | R. Vilanova   |
|  2 |          1 | S. Alcántara  |
|  5 |          2 | A. Visioli    |
|  4 |          2 | F. Padula     |
|  6 |          2 | S. Alcántara  |
|  7 |          3 | S. Skogestad  |
+----+------------+---------------+
```

Without further ado, here is how the migration looks like:

```python
app_name = 'app'
model_name = 'author'
related_model_name = 'article'
model_table = '%s_%s' % (app_name, model_name)
pivot_table = '%s_%s_%ss' % (app_name, related_model_name, model_name)
fk_name, index_name = None, None 


class Migration(migrations.Migration):
    ...

    operations = [
        migrations.AddField(
            model_name=model_name,
            name='id',
            field=models.IntegerField(null=True),
            preserve_default=True,
        ),
        migrations.RunPython(do_most_of_the_surgery),
        migrations.AlterField(
            model_name=model_name,
            name='id',
            field=models.AutoField(
                verbose_name='ID', serialize=False, auto_created=True,
                primary_key=True),
            preserve_default=True,
        ),
        migrations.AlterField(
            model_name=model_name,
            name='name',
            field=models.CharField(max_length=30, unique=True),
            preserve_default=True,
        ),
        migrations.RunPython(do_the_final_lifting),
    ]
```

In summary, the migration goes through the following operations:

* Declare the new `id` column in the authors table nullable
* Run the `do_most_of_the_surgery` function
* Declare the `id` field as the new (autoincrement) PK column
* Alter the former PK field (`name`) accordingly
* Do some extra final work by calling `do_the_final_lifting`

Let us turn our attention to the `do_most_of_the_surgery` function now:

```python
def do_most_of_the_surgery(apps, schema_editor):
    models = {}
    Model = apps.get_model(app_name, model_name)

    # Generate values for the new id column
    for i, o in enumerate(Model.objects.all()):
        o.id = i + 1
        o.save()
        models[o.name] = o.id

    # Work on the pivot table before going on
    drop_constraints_and_indices_in_pivot_table()

    # Drop current pk index and create the new one
    cursor.execute(
        "ALTER TABLE %s DROP PRIMARY KEY" % model_table
    )
    cursor.execute(
        "ALTER TABLE %s ADD PRIMARY KEY (id)" % model_table
    )

    # Rename the fk column in the pivot table
    cursor.execute(
        "ALTER TABLE %s "
        "CHANGE %s_id %s_id_old %s NOT NULL" %
        (pivot_table, model_name, model_name, 'VARCHAR(30)'))
    # ... and create a new one for the new id
    cursor.execute(
        "ALTER TABLE %s ADD COLUMN %s_id INT(11)" %
        (pivot_table, model_name))

    # Fill in the new column in the pivot table
    cursor.execute("SELECT id, %s_id_old FROM %s" % (model_name, pivot_table))
    for row in cursor:
        id, key = row[0], row[1]
        model_id = models[key]

        inner_cursor = connection.cursor()
        inner_cursor.execute(
            "UPDATE %s SET %s_id=%d WHERE id=%d" %
            (pivot_table, model_name, model_id, id))

    # Drop the old (renamed) column in pivot table, no longer needed
    cursor.execute(
        "ALTER TABLE %s DROP COLUMN %s_id_old" %
        (pivot_table, model_name))
```

where

```python
def drop_constraints_and_indices_in_pivot_table():
    global fk_name, index_name

    fk_postfix = '%%_fk_%s_%s_%s' % (app_name, model_name, 'name')
    cursor.execute(
        "SELECT constraint_name FROM information_schema.table_constraints "
        "WHERE table_name='%s' AND constraint_name LIKE "
        "'%s'" % (pivot_table, fk_postfix))
    fk_name = cursor.fetchone()[0]

    cursor.execute(
        "SELECT index_name FROM information_schema.statistics "
        "WHERE table_name='%s' AND index_name LIKE "
        "'%s_%%' AND column_name='%s_id'" %
        (pivot_table, pivot_table, model_name))
    index_name = cursor.fetchone()[0]

    cursor.execute(
        "ALTER TABLE %s DROP FOREIGN KEY %s" % (pivot_table, fk_name))
    cursor.execute(
        "ALTER TABLE %s DROP INDEX %s" % (pivot_table, index_name))
    cursor.execute(
        "ALTER TABLE %s DROP INDEX %s_id" % (pivot_table, related_model_name))
```

As you can see, there is nothing fancy here, just a lot of unpleasant work to fix a major pitfall.
For completeness, here is the code for `do_the_final_lifting` too:

```python
def do_the_final_lifting(apps, schema_editor):
    # Create a new unique index for the old pk column
    index_prefix = '%s_id' % model_table
    new_index_prefix = '%s_name' % model_table
    new_index_name = index_name.replace(index_prefix, new_index_prefix)

    cursor.execute(
        "ALTER TABLE %s ADD UNIQUE KEY %s (%s)" %
        (model_table, new_index_name, 'name'))

    # Finally, work on the pivot table
    recreate_constraints_and_indices_in_pivot_table()
```

where 

```python
def recreate_constraints_and_indices_in_pivot_table():
    cursor.execute(
        "ALTER TABLE %s MODIFY %s_id INT(11) NOT NULL" %
        (pivot_table, model_name))
    cursor.execute(
        "ALTER TABLE %s ADD INDEX %s (%s_id)" %
        (pivot_table, index_name, model_name))

    fk_postfix = '_fk_%s_%s_%s' % (app_name, model_name, 'name')
    new_fk_postfix = '_fk_%s_%s_%s' % (app_name, model_name, 'id')
    new_fk_name = fk_name.replace(fk_postfix, new_fk_postfix)

    cursor.execute(
        "ALTER TABLE %s ADD CONSTRAINT "
        "%s FOREIGN KEY (%s_id) REFERENCES %s (id)" %
        (pivot_table, new_fk_name, model_name, model_table))
    cursor.execute(
        "ALTER TABLE %s ADD UNIQUE KEY %s_id (%s_id, %s_id)" %
        (pivot_table, related_model_name, related_model_name, model_name))
```

That's all! Hope someone finds it useful :wink:. Any feedback is welcome, so feel free to leave your comments.

[^1]: implicitly, Django adds the following line

    ```python
    id = models.AutoField(primary_key=True)
    ```

    to the model class.

[^2]: see https://stackoverflow.com/questions/29657312/update-primary-key-django-mysql.

[^3]: see, for example:

    * https://stackoverflow.com/questions/2055784/what-is-the-best-approach-to-change-primary-keys-in-an-existing-django-app
    * https://stackoverflow.com/questions/41185561/how-to-migrate-from-custom-primary-key-to-default-id
    
    to convice yourself.