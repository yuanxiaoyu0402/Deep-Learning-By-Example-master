# Deep-Learning-By-Example-master
# 判别器和生成器损耗

在这一部分中, 我们需要定义判别器和生成器损耗, 这可以被认为是此实现中最棘手的部分。
我们知道, 生成器试图复制原始图像。判别器像一名法官那样工作, 从生成器接收图像和原始输入图像。因此, 在为每个部分设计我们的损失时, 我们需要针对两件事情。
首先, 我们需要网络的判别器部分能够区分生成器生成的假图像和来自原始的培训实例的真正图像。在训练期间, 我们将用一批被分成两类的数据输入给判别器。第一个类别是来自原始输入的图像。第二类是生成器生成的假图像。
因此, 判别器的最终一般损失将是其接受真实作为真实的能力,和检测到假的是假的的能力;那么最终的全部损失将是:

```python
disc_loss=disc_loss_real+disc_loss_fake
tf.reduce_mean(
    tf.nn.sigmoid_cross_entropy_with_logits(
        labels=tf.ones_like(disc_logits_fake),
        logits=disc_logits_fake))
```

因此, 我们需要计算两个损失, 才能得出最终的判别损失。        
第一种损失,disc_loss_real,从判别器和标签中得到基于logits的计算。在这种情况下, 这个迷你批次中的所有图像都来自mmist 数据集的真正的输入图像。为了提高模型对测试集的泛化能力, 并给出更好的结果, 人们发现, 实际改变值1到0.9 更好。
这种对标签的更改引入了一种称为标签平滑的内容: 

```python
disc_labels_real = tf.ones_like(disc_logits_real) * (1 - label_smooth)
```

对于第二部分的判别器损耗, 即判别器检测假图像的能力。我们将从判别器和标签中得到日志值的损失;所有的这些都是零, 因为我们知道, 这个迷你批次的所有图像来自生成器, 而不是来自原始输入。
现在我们已经讨论了判别器损耗, 我们需要计算发生器的损耗。生成器损耗将被称为gen_loss,这将是disc_logits_fake(判别器判别假图像的输出)和标签（这将是所有的, 因为发生器试图说服判别器与其假图像的设计是真的）之间的损失：

```python
# calculating the losses of the discrimnator and generator
disc_labels_real = tf.ones_like(disc_logits_real) * (1 - label_smooth)
disc_labels_fake = tf.zeros_like(disc_logits_fake)

disc_loss_real = tf.nn.sigmoid_cross_entropy_with_logits(labels=disc_labels_real, logits=disc_logits_real)
disc_loss_fake = tf.nn.sigmoid_cross_entropy_with_logits(labels=disc_labels_fake, logits=disc_logits_fake)

#averaging the disc loss
disc_loss = tf.reduce_mean(disc_loss_real + disc_loss_fake)

#averaging the gen loss
gen_loss = tf.reduce_mean(
    tf.nn.sigmoid_cross_entropy_with_logits(
        labels=tf.ones_like(disc_logits_fake),
        logits=disc_logits_fake))
```

# 优化器

最后, 优化器部分!在本节中, 我们将定义优化标准, 这些标准将在培训过程中使用。首先, 我们将分别更新生成器和判别器, 所以我们需要能够检索的变量每个部分。
对于第一个优化器, 生成的, 我们将检索所有的变量, 从generator，来自于计算图的可训练变量开始。然后我们就可以通过引用它的名称来检查哪个变量是哪个变量。
我们将做同样的鉴别变量, 通过鉴别所有变量，从discriminator开始。之后, 我们可以传递我们想要优化的变量列表到优化器。
因此, TensorFlow的可变作用域特征使我们能够从某个字符串开始检索变量,然后我们可以有两个不同的变量列表, 一个用于发生器和另一个用于鉴别器:

```python
# building the model optimizer

learning_rate = 0.002

# Getting the trainable_variables of the computational graph, split into Generator and Discrimnator parts
trainable_vars = tf.trainable_variables()
gen_vars = [var for var in trainable_vars if var.name.startswith("generator")]
disc_vars = [var for var in trainable_vars if var.name.startswith("discriminator")]

disc_train_optimizer = tf.train.AdamOptimizer().minimize(disc_loss, var_list=disc_vars)
gen_train_optimizer = tf.train.AdamOptimizer().minimize(gen_loss, var_list=gen_vars)
```

# 模型训练

现在让我们开始训练过程, 看看如何管理生成类似于 mnist 的图像:

```python
train_batch_size = 100
num_epochs = 100
generated_samples = []
model_losses = []

saver = tf.train.Saver(var_list=gen_vars)

with tf.Session() as sess:
    sess.run(tf.global_variables_initializer())

    for e in range(num_epochs):
        for ii in range(mnist_dataset.train.num_examples // train_batch_size):
            input_batch = mnist_dataset.train.next_batch(train_batch_size)

            # Get images, reshape and rescale to pass to D
            input_batch_images = input_batch[0].reshape((train_batch_size, 784))
            input_batch_images = input_batch_images * 2 - 1

            # Sample random noise for G
            gen_batch_z = np.random.uniform(-1, 1, size=(train_batch_size, gen_z_size))

            # Run optimizers
            _ = sess.run(disc_train_optimizer,
                         feed_dict={real_discrminator_input: input_batch_images, generator_input_z: gen_batch_z})
            _ = sess.run(gen_train_optimizer, feed_dict={generator_input_z: gen_batch_z})

        # At the end of each epoch, get the losses and print them out
        train_loss_disc = sess.run(disc_loss,
                                   {generator_input_z: gen_batch_z, real_discrminator_input: input_batch_images})
        train_loss_gen = gen_loss.eval({generator_input_z: gen_batch_z})

        print("Epoch {}/{}...".format(e + 1, num_epochs),
              "Disc Loss: {:.3f}...".format(train_loss_disc),
              "Gen Loss: {:.3f}".format(train_loss_gen))

        # Save losses to view after training
        model_losses.append((train_loss_disc, train_loss_gen))

        # Sample from generator as we're training for viegenerator_inputs_zwing afterwards
        gen_sample_z = np.random.uniform(-1, 1, size=(16, gen_z_size))
        generator_samples = sess.run(
            generator(generator_input_z, input_img_size, reuse_vars=True),
            feed_dict={generator_input_z: gen_sample_z})

        generated_samples.append(generator_samples)
        saver.save(sess, './checkpoints/generator_ck.ckpt')

# Save training generator samples
with open('train_generator_samples.pkl', 'wb') as f:
    pkl.dump(generated_samples, f)
```

在运行了100个时代的模型后, 我们有了一个训练有素的模型, 将能够生成与我们提供给判别器的原始输入图像类似的图像:

```python
fig, ax = plt.subplots()
model_losses = np.array(model_losses)
plt.plot(model_losses.T[0], label='Disc loss')
plt.plot(model_losses.T[1], label='Gen loss')
plt.title("Model Losses")
plt.legend()
```

输出：
如上图所示, 你可以看到模型损失, 这是表示判别器和发生器的曲线是收敛的。
