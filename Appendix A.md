# 附录 A
本附录介绍了 ```TypeScript```（使用的版本 ```4.7.4```）中对象验证库的实现。 这个库使用 ```validated_types``` ```NPM``` 库作为基础。

此对象验证库将能够验证 ```JSON``` 对象，其架构以以下格式给出。 模式对象列出了要验证的对象的属性。 每个属性的值是要验证的对象中该属性的模式。 在此示例中，我们实现了对验证整数、字符串和嵌套对象的支持。 还可以添加其他类型（例如，浮点数和数组）的验证。

整数属性的架构定义如下：

```json
['int' | 'int?',  'min-value,max-value']
```

字符串属性的模式使用如下所示的三种语法之一定义：

```json
['string' | 'string?',  'min-length,max-length,validator']

['string' | 'string?',  ['min-length,max-length,validator',
                         'validator']]
            
             
['string' | 'string?',  ['min-length,max-length,validator',
                         'validator',
                         'validator']]
```

下面是一个示例架构对象：

```typescript
const schema = {
  inputUrl: ['string', ['1,1024,url',
                        'startsWith,https',
                        'endsWith,com']] as const,
                         
  outputPort: ['int?', '1,65535'] as const,
  mongoDb: {
    user: ['string?', '1,512'] as const
  },
  transformations: [
    {
      outputFieldName: ['string', '1,512'] as const,
      inputFieldName: ['string', '1,512'] as const
    }
  ] as const
};

const defaultValues = {
  outputPort: 8080
};
```

上面对单个属性模式的解释：

- ```inputUrl```
  强制字符串属性
  长度必须介于 ```1-1024``` 个字符之间
  值必须是一个 ```URL```，以```https```开头，以```com```结尾
- ```outputPort```
  可选的整数属性
  值必须介于 ```1-65535``` 之间
- ```mongoDb.user```
  可选字符串属性
  长度必须介于 ```1-512``` 个字符之间
- ```transformations[*].outputFieldName```
  强制字符串属性
  长度必须介于 ```1-512``` 个字符之间
- ```transformations[*].inputFieldName```
  强制字符串属性
  长度必须介于 ```1-512``` 个字符之间

以下是库所需的 ```TypeScript``` 类型的定义：

```ValidatedObject.ts```

```typescript
import { VInt, VString } from 'validated-types';

type Type = string;
type ValidationSpec = string | [string, string] | [string, string, string];

type ValidationClass<T extends Type, V extends ValidationSpec> = T extends `${infer NullableType}?`
  ? NullableType extends 'string'
    ? VString<V> | null
    : NullableType extends 'int'
    ? V extends string
      ? VInt<V> | null
      : never
    : never
  : T extends `${infer T}`
  ? T extends 'string'
    ? VString<V>
    : T extends 'int'
    ? V extends string
      ? VInt<V>
      : never
    : never
  : never;

type PropertySchema = [Type, ValidationSpec];
export type ObjectSchema = { [propertyName: string]: PropertySchema | ObjectSchema | [ObjectSchema] };

export type ValidatedObject<S extends ObjectSchema> = {
  [P in keyof S]: S[P] extends [ObjectSchema]
    ? Array<ValidatedObject<S[P][0]>>
    : S[P] extends ObjectSchema
    ? ValidatedObject<S[P]>
    : S[P] extends PropertySchema
    ? ValidationClass<S[P][0], S[P][1]>
    : never;
};
```

下面的 ```tryCreateValidatedObject``` 函数尝试创建给定对象的验证版本：

```tryCreateValidatedObject.ts```

```typescript
mport { DeepWritable, Writable } from 'ts-essentials';
import { ObjectSchema, ValidatedObject } from './ValidatedObject';

function tryCreateValidatedObject<S extends ObjectSchema>(
  object: Record<string, any>,
  objectSchema: OS,
  defaultValuesObject?: Record<string, any>
): ValidatedObject<S> {
  const validatedObject = {};

  Object.entries(objectSchema as Writable<S>).forEach(([propertyName,
                                                        objectOrPropertySchema]) => {
    const isPropertySchema = Array.isArray(objectOrPropertySchema) &&
                             typeof objectOrPropertySchema[0] === 'string';

    if (isPropertySchema) {
      const [propertyType, propertySchema] = objectOrPropertySchema;
      if (propertyType.startsWith('string')) {
        (validatedObject as any)[propertyName] = propertyType.endsWith('?')
          ? VString.create(propertySchema, object[propertyName] ??
              defaultValuesObject?.[propertyName])
          : VString.tryCreate(propertySchema, object[propertyName]);
      } else if (propertyType.startsWith('int') && !Array.isArray(propertySchema)) {
        (validatedObject as any)[propertyName] = propertyType.endsWith('?')
          ? VInt.create<typeof propertySchema>(
              propertySchema,
              object[propertyName] ?? defaultValuesObject?.[propertyName]
            )
          : VInt.tryCreate<typeof propertySchema>(propertySchema, object[propertyName]);
      }
    } else {
      if (Array.isArray(objectOrPropertySchema)) {
        (object[propertyName] ?? []).forEach((subObject: any) => {
          (validatedObject as any)[propertyName] = [
            ...((validatedObject as any)[propertyName] ?? []),
            tryCreateValidatedObject(subObject ?? {},
                                     objectOrPropertySchema[0],
                                     defaultValuesObject?.[propertyName])
          ];
        });
      } else {
        (validatedObject as any)[propertyName] = tryCreateValidatedObject(
          object[propertyName] ?? {},
          objectOrPropertySchema,
          defaultValuesObject?.[propertyName]
        );
      }
    }
  });

  return validatedObject as ValidatedObject<ObjSchema>;
}
```

让我们拥有以下未验证的配置对象并创建它的验证版本：

```configuration.ts```

```typescript
const unvalidatedConfiguration = {
  inputUrl: 'https://www.google.com',
  mongoDb: {
    user: 'root'
  },
  transformations: [
    {
      outputFieldName: 'outputField',
      inputFieldName: 'inputField'
    }
  ]
};

// This will contain the validated configuration object with strong typing
let configuration: ValidatedObject<DeepWritable<typeof configurationSchema>>;

try {
  configuration = tryCreateValidatedObject(
    unvalidatedConfiguration,
    configurationSchema as DeepWritable<typeof configurationSchema>,
    configurationDefaultValues
  );

  console.log(validatedConfiguration.inputUrl.value);
  console.log(validatedConfiguration.outputPort?.value);
  console.log(validatedConfiguration.mongoDb.user?.value);
  console.log(validatedConfiguration.transformations[0].inputFieldName.value);
  console.log(validatedConfiguration.transformations[0].outputFieldName.value);
} catch (error) {
  // Handle error...
}

// We export the validated configuration object with strong typing
// to be used in other parts of the application
export default configuration;

```

下面是 ```Node.js process.env``` 对象的示例模式：

```environmentSchema.ts```

```typescript
export const environmentSchema = {
  NODE_ENV: ['string', '1,32,isOneOf,["development","integration","production"]'],
  LOGGING_LEVEL: ['string?', '1,32,isOneOf,["DEBUG","INFO","WARN","ERROR"]'],
  MONGODB_PORT: ['string?', '1,5,numericRange,1-65535'],
  MONGODB_USER: [string?, '1,512']
}

export const environmentDefaultValues = {
  LOGGING_LEVEL: 'INFO',
  MONGODB_PORT: '27017'
}
```

我们可以从 ```process.env``` 创建一个经过验证的环境：

```environment.ts```

```typescript
import { environmentDefaultValue, environmentSchema } from './environmentSchema';

let environment: ValidatedObject<DeepWritable<typeof environmentSchema>>;

try {
  environment = tryCreateValidatedObject(
    process.env,
    environmentSchema as DeepWritable<typeof environmentSchema>,
    environmentDefaultValues
  );
} catch (error) {
  // Handle error...
}

export default environment;