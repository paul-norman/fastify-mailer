# fastify-mailer

This is a fork of [Jean-Michel Coghe's plugin](https://github.com/coopflow/fastify-mailer) with very minor tweaks to better suit my needs.

It offers [Nodemailer](https://www.nodemailer.com) instance initialisation and encapsulation for the [fastify](https://www.github.com/fastify/fastify) framework.

## Install

Install the package with:

```sh
npm i github:paul-norman/fastify-mailer
```

## Usage

The package needs to be added to your project with `register` and you must configure your transporter options following [Nodemailer documentation](https://nodemailer.com/usage/).

`transport` is a *required* transport configuration object, connection url or a transport plugin instance.

`defaults` is an *optional* object containing message default values.

```js
'use strict';

import Fastify       from 'fastify';
import FastifyMailer from 'paul-norman/fastify-mailer';

const fastify = Fastify({ 
	logger: true
});

// Register the plugin
fastify.register(FastifyMailer, {
	transport: {
		host:   'smtp.example.tld',
		port:   465,
		secure: true, // use TLS
		auth: {
			user: 'john.doe',
			pass: 'super strong password'
		}
	},
	// Optionally define defaults to be used
	defaults: { 
		from:    'John Doe <john.doe@example.tld>',
		subject: 'default example',
	},
});

fastify.get('/send', async (request, reply) => {
	const { mailer } = fastify;
	
	// Async / Await
	try {
		let info = await mailer.sendMail({
			to:      'someone@example.tld',
			subject: 'example',
			text:    'hello world!'
		});

		reply.status(200);
		return {
			status:  'ok',
			message: 'Email successfully sent',
			info: {
				from: info.from, // John Doe <john.doe@example.tld>
				to:   info.to,   // ['someone@example.tld']
			}
		};
	} catch (error) {
		fastify.log.error(errors);
			
		reply.status(500);
		return {
			status:  'error',
			message: 'Something went wrong'
		}
	}
	
	// Callback
	mailer.sendMail({
		to:      'someone@example.tld',
		subject: 'example',
		text:    'hello world!'
	}, (errors, info) => {
		if (errors) {
			fastify.log.error(errors);
			
			reply.status(500);
			return {
				status:  'error',
				message: 'Something went wrong'
			}
		}
	
		reply.status(200);
		return {
			status:  'ok',
			message: 'Email successfully sent',
			info: {
				from: info.from, // John Doe <john.doe@example.tld>
				to:   info.to,   // ['someone@example.tld']
			}
		};
	});
});

const start = async () => {
	try {
		await fastify.listen({
			host: '0.0.0.0',
			port: 3000,
		});
	} catch (error) {
		fastify.log.error(error);
		process.exit(1);
	}
};
start();
```

### Namespaced example:

`namespace`: is an *optional* string that lets you define multiple namespaced transporter instances _(with different options parameters if you wish)_ that you can later use in your application.

```js
'use strict';

import Fastify       from 'fastify';
import FastifyMailer from 'paul-norman/fastify-mailer';

const fastify = Fastify({ 
	logger: true
});

fastify
	.register(require('fastify-mailer'), {
		defaults: {
			from:    'Jane Doe <jane.doe@example.tld>',
			subject: 'default subject',
		},
		namespace: 'jane',
		transport: {
			host: 'smtp.example.tld',
			port: 465,
			secure: true, // use TLS
			auth: {
				user: 'jane.doe',
				pass: 'super strong password for jane'
			}
		}
	})
	.register(require('fastify-mailer'), {
		defaults: {
			from: 'John Doe <john.doe@example.tld>'
		},
		namespace: 'john',
		transport: {
			pool: true,
			host: 'smtp.example.tld',
			port: 587,
			secure: false,
			auth: {
				user: 'john.doe',
				pass: 'super strong password for john'
			}
		}
	});

fastify.get('/send_from_jane', (request, reply) => {
	const { mailer } = fastify;
	
	mailer.jane.sendMail({
		to:      'someone@example.tld',
		subject: 'example from Jane',
		text:    'hello from Jane!'
	}, (errors, info) => {
		if (errors) {
			fastify.log.error(errors);
		
			reply.status(500);
			return {
				status:  'error',
				message: 'Something went wrong'
			};
		}
		
		reply.status(200);
		return {
			status:  'ok',
			message: 'Email successfully sent',
			info: {
				from: info.from, // Jane Doe <jane.doe@example.tld>
				to:   info.to,   // ['someone@example.tld']
			}
		};
	});
});

fastify.get('/send_from_john', (request, reply) => {
	const { mailer } = fastify;
	
	mailer.john.sendMail({
		to:      'someone@example.tld',
		subject: 'example from Jane',
		text:    'hello from Jane!'
	}, (errors, info) => {
		if (errors) {
			fastify.log.error(errors);
		
			reply.status(500);
			return {
				status:  'error',
				message: 'Something went wrong'
			};
		}
		
		reply.status(200);
		return {
			status:  'ok',
			message: 'Email successfully sent',
			info: {
				from: info.from, // John Doe <john.doe@example.tld>
				to:   info.to,   // ['someone@example.tld']
			}
		};
	});
});

const start = async () => {
	try {
		await fastify.listen({
			host: '0.0.0.0',
			port: 3000,
		});
	} catch (error) {
		fastify.log.error(error);
		process.exit(1);
	}
};
start();
```

### Example using SES transport:

```js
'use strict';

import Fastify       from 'fastify';
import FastifyMailer from 'paul-norman/fastify-mailer';
import aws           from '@aws-sdk/client-ses';

const fastify = Fastify({ 
	logger: true
});

/**
 * Configure AWS SDK:
 *
 * Use environment variables or Secrets as a Service solutions 
 * to store your secrets.
 *
 * NB: DO NOT hardcode your secrets, this is only an example!
 */
process.env.AWS_ACCESS_KEY_ID = 'aws_access_key_id_here'
process.env.AWS_SECRET_ACCESS_KEY = 'aws_secret_access_key_here'

const ses = new aws.SES({
	apiVersion: '2010-12-01',
	region: 'us-east-1'
});

fastify.register(FastifyMailer, {
	defaults: { 
		from: 'John Doe <john.doe@example.tld>'
	},
	transport: {
		SES: { 
			ses,
			aws
		}
	}
});

fastify.get('/send', (request, reply) => {
	const { mailer } = fastify;
	
	mailer.sendMail({
		to:      'someone@example.tld',
		subject: 'example',
		text:    'hello world!',
		ses: {
			// optional extra arguments for SendRawEmail
			Tags: [
				{
					Name:  'foo',
					Value: 'bar'
				}
			]
		}
	}, (errors, info) => {
		if (errors) {
			fastify.log.error(errors);
		
			reply.status(500);
			return {
				status:  'error',
				message: 'Something went wrong'
			}
		}
	
		reply.status(200);
		return {
			status:  'ok',
			message: 'Email successfully sent',
			info: {
				envelope: info.envelope, // {"from":"John Doe <john.doe@example.tld>","to":['someone@example.tld']}
			}
		}
	});
});

const start = async () => {
	try {
		await fastify.listen({
			host: '0.0.0.0',
			port: 3000,
		});
	} catch (error) {
		fastify.log.error(error);
		process.exit(1);
	}
};
start();
```

For more information on transports you can take a look at [Nodemailer dedicated documentation](https://nodemailer.com/transports/).

## Typescript users

Types for nodemailer are not officially supported by its author Andris Reinman.

If you want to use the DefinitelyTyped community maintained types, install the package with:

```shell
npm install -D @types/nodemailer
```

then re-declare the `mailer` interface in the `fastify` module within your own code to add the properties you expect.

### Example:

```ts
import { Transporter } from 'nodemailer';

export interface FastifyMailerNamedInstance {
	[namespace: string]: Transporter;
}
export type FastifyMailer = FastifyMailerNamedInstance & Transporter;

declare module 'fastify' {
	interface FastifyInstance {
		mailer: FastifyMailer;
	}
}
```

## Documentation

See [Nodemailer documentation](https://nodemailer.com/about/).

## Acknowledgements

This project is a fork of [coopflow's plugin](https://github.com/coopflow/fastify-mailer).

## License

Licensed under [MIT](https://github.com/coopflow/fastify-mailer/blob/master/LICENSE)