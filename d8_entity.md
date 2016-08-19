<h2>Building a bundle-less content entity type in Drupal 8.
</h2>

In this case we create a drupal 8 content entity that does not have any bundles.

The entity does not implement the Field API so it remains in code throughout. Nevertheless it can be a useful skeleton for building out Content Entities as we import more complex data later. 

Finally, where there are some OOP concepts, I will refer to the relevant docs. 

<h3>Background.</h3>
Our module is called advertiser.
Our content entity type is called advertiser.

Our new Drupal 8 advertiser content entity will have basetable fields:
  - UUID
  - ID

<h3>Defining a Content Entity in Drupal 8.</h3>

First all our custom entities now live at <code>web/modules/custom</code>.

Inside our custom entity the file structure that we will end up with looks like this:
<code>web/modules/custom/advertiser$
├── advertiser.info.yml
└── src
    └── Entity
         └── Advertiser.php
</code>

For reference here is the finished entity along with added extras like tests and constraint plugins: https://github.com/gl2748/advertiser_entity, but let's keep things simple for the time being.

We start by defining our custom module in <code>module_name.info.yml</code>. This is self explanatory:

<code>
name: Advertiser
type: module
description: 'Barebones advertiser entity'
package: custom
core: 8.x
</code>

Meanwhile the basic Advertiser entity class and associated schema is defined in <code>src/Entity/Advertiser.php</code>.

The first thing we do is define a <a href="http://php.net/namespace">namespace</a> for our Advertiser Entity Class. This will come in handy whenever we want to use our classes.
<code>namespace Drupal\advertiser\Entity</code>; 

Now it is time to define our entity, which we do in an annotation. <strong>TIP:</strong> This is the actual definition of the entity type it is read and cached so be sure to clear the cache after any changes. 

<?php
/**
 * Defines the Advertiser entity.
 *
 * @ingroup advertiser
 *
 * @ContentEntityType(
 *   id = "advertiser",
 *   label = @Translation("Advertiser"),
 *   base_table = "advertiser",
 *   entity_keys = {
 *     "id" = "id",
 *     "uuid" = "uuid",
 *   },
 * )
 */
?>

Because this is a barebones entity we only use a few properties and no handlers (like access). For a comprehensive list of available options for your entity see the docs here: https://www.drupal.org/node/2192175

At this point we have a functional module that defines our custom content entity, however if we go ahead and enable the module we will see that the 'advertiser' table has not been created in the database.

<code>$ drush sqlc
mysql> SHOW TABLES;</code>

This is because our class doesn't have any methods that explictly interact with the database. Furthermore we need a description of the bare minimum of methods needed for an entity to interface satisfactorily with the Database. 

Generally we can add classes by adding something like <code>use Drupal\Core\Entity\ContentEntityBase;</code> after our namespace definition at the top of our script. This makes these methods available to our own class, which can <em><strong>extend</strong></em> them or in the case of Interfaces, <em><strong>implement</strong></em> them.

We do two things, we <strong>extend</strong> an existing <a href="https://api.drupal.org/api/drupal/core!lib!Drupal!Core!Entity!ContentEntityBase.php/class/ContentEntityBase/8"><strong>ContentEntityBase</strong></a> class that already has the necessary methods to interact with the DB, and <strong>implement</strong> an <a href="https://api.drupal.org/api/drupal/core!lib!Drupal!Core!Entity!ContentEntityInterface.php/interface/ContentEntityInterface/8"><strong>ContentEntityInterface</strong></a> to describe...

<blockquote>the methods that we need to access our database. It does NOT describe in any way HOW we achieve that. That's what the IMPLEMENTing class does. We can IMPLEMENT this interface as many times as we need in as many different ways as we need. We can then switch between implementations of the interface without impact to our code because the interface defines how we will use it regardless of how it actually works. - https://secure.php.net/manual/en/language.oop5.interfaces.php</blockquote>

All this means is that we end up with the following: <strong>Tip</strong> Remember to add any new classes through a <code>use</code> statement at the top of our script:

<code>class Advertiser extends ContentEntityBase implements ContentEntityInterface {</code>

But we still need use these new useful methods to put something in the database, we start with the basic fields for our entity.

The method <a href="https://api.drupal.org/api/drupal/core!lib!Drupal!Core!Entity!FieldableEntityInterface.php/function/FieldableEntityInterface%3A%3AbaseFieldDefinitions/8">baseFieldDefinitions</a> comes from <a href="https://api.drupal.org/api/drupal/core!lib!Drupal!Core!Entity!ContentEntityBase.php/class/ContentEntityBase/8">ContentEntityBase</a> class that we are extending.  
It takes one parameter:
<blockquote>The entity type definition. Useful when a single class is used for multiple, possibly dynamic entity types.
</blockquote>
And it returns
<blockquote>An array of base field definitions for the entity type, keyed by field name.</blockquote>

So we implement it like this:

<?php
public static function baseFieldDefinitions(EntityTypeInterface $entity_type) {
      
    // Standard field, used as unique if primary index.
    $fields['id'] = BaseFieldDefinition::create('integer')
      ->setLabel(t('ID'))
      ->setDescription(t('The ID of the Contact entity.'))
      ->setReadOnly(TRUE);

    // Standard field, unique outside of the scope of the current project.
    $fields['uuid'] = BaseFieldDefinition::create('uuid')
      ->setLabel(t('UUID'))
      ->setDescription(t('The UUID of the Contact entity.'))
      ->setReadOnly(TRUE);
      
    return $fields;
  }
?>

It's worth noting that:  

<a href="https://api.drupal.org/api/drupal/core!lib!Drupal!Core!Entity!FieldableEntityInterface.php/function/FieldableEntityInterface%3A%3AbaseFieldDefinitions/8"><strong>BaseFieldDefinitions</strong></a>
<blockquote>"Provides base field definitions for an entity type."</blockquote> - It is a public static method from the <a href="https://api.drupal.org/api/drupal/core!lib!Drupal!Core!Entity!FieldableEntityInterface.php/function/FieldableEntityInterface%3A%3AbaseFieldDefinitions/8">FieldableEntityInterface</a>Interface.
 
<strong><a href="https://api.drupal.org/api/drupal/core!lib!Drupal!Core!Field!BaseFieldDefinition.php/class/BaseFieldDefinition/8">BaseFieldDefinition</a></strong> <blockquote>"A class for defining entity fields."</blockquote> - All the methods we need to <a href="https://api.drupal.org/api/drupal/core!lib!Drupal!Core!Field!BaseFieldDefinition.php/function/BaseFieldDefinition%3A%3Acreate/8">create</a> fields, add <a href="https://api.drupal.org/api/drupal/core!lib!Drupal!Core!Field!BaseFieldDefinition.php/function/BaseFieldDefinition%3A%3AaddPropertyConstraints/8">constraints</a>, etc...  

So altogether we have this:

<?php

namespace Drupal\advertiser\Entity;

use Drupal\Core\Entity\ContentEntityBase;
use Drupal\Core\Field\BaseFieldDefinition;
use Drupal\Core\Entity\EntityTypeInterface;
use Drupal\Core\Entity\ContentEntityInterface;

/**
 * Defines the advertiser entity.
 *
 * @ingroup advertiser
 *
 * @ContentEntityType(
 *   id = "advertiser",
 *   label = @Translation("advertiser"),
 *   base_table = "advertiser",
 *   entity_keys = {
 *     "id" = "id",
 *     "uuid" = "uuid",
 *   },
 * )
 */

class Advertiser extends ContentEntityBase implements ContentEntityInterface {

  public static function baseFieldDefinitions(EntityTypeInterface $entity_type) {

    // Standard field, used as unique if primary index.
    $fields['id'] = BaseFieldDefinition::create('integer')
      ->setLabel(t('ID'))
      ->setDescription(t('The ID of the Contact entity.'))
      ->setReadOnly(TRUE);

    // Standard field, unique outside of the scope of the current project.
    $fields['uuid'] = BaseFieldDefinition::create('uuid')
      ->setLabel(t('UUID'))
      ->setDescription(t('The UUID of the Contact entity.'))
      ->setReadOnly(TRUE);

    return $fields;
  }
}
?>

Now when we run <code>drush en advertiser -y; drush sqlc</code> we can see the 'advertiser' table has been added to the database!

