#include <linux/init.h>         
#include <linux/module.h>         
#include <linux/device.h>       
#include <linux/kernel.h>     
#include <linux/fs.h>            
#include <linux/uaccess.h>  
#include <linux/wait.h>       

#define  DEVICE_NAME "viktoria"    
#define  CLASS_NAME  "vik"        
#define SIZE_BUF 256 
 
MODULE_LICENSE("GPL");                 
 
static int Major;                  
static char message[256] = {0};          
static short size_of_message;           
static int numberOpens = 0;              
static struct class* devClass  = NULL; 
static struct device* devDevice = NULL; 

static char Buf[BUFFER_SIZE];
static int write_ count = 1;
static int read_count = 0;

static char flag = 'n';
//static DECLARE_WAIT_QUEUE_HEAD(wq);
static wait_queue_head_t wq_write;
static wait_queue_head_t wq_read;


static int dev_open(struct inode *, struct file *);
static int dev_release(struct inode *, struct file *);
static ssize_t dev_read(struct file *, char *, size_t, loff_t *);
static ssize_t dev_write(struct file *, const char *, size_t, loff_t *);

static struct file_operations fops =
{
   .open = dev_open,
   .read = dev_read,
   .write = dev_write,
   .release = dev_release,
};

static int dev_init(void){

   init_waitqueue_head(&wq_write);
   init_waitqueue_head(&wq_read);
   printk(KERN_INFO "Initializing the EBBChar LKM\n");

   Major = register_chrdev(0, DEVICE_NAME, &fops); 
   if (Major<0){
      printk(KERN_ALERT "Registering char device failed with %d\n", Major);
      return Major;
   }
   printk(KERN_INFO "Registered correctly with major number %d\n", Major);

   devClass = class_create(THIS_MODULE, CLASS_NAME);
   if (IS_ERR(devClass)){                
      unregister_chrdev(Major, DEVICE_NAME);
      printk(KERN_ALERT "Failed to register device class\n");
      return PTR_ERR(devClass);          
   }
   printk(KERN_INFO "Device class registered correctly\n");

   devDevice = device_create(devClass, NULL, MKDEV(Major, 0), NULL, DEVICE_NAME);
   if (IS_ERR(devDevice)){               
      class_destroy(devClass);           
      unregister_chrdev(Major, DEVICE_NAME);
      printk(KERN_ALERT "Failed to create the device\n");
      return PTR_ERR(devDevice);
   }
   printk(KERN_INFO "Device class created correctly\n"); 
   return 0;
}

static void dev_exit(void){
   device_destroy(devClass, MKDEV(Major, 0));     
   class_unregister(devClass);                         
   class_destroy(devClass);                           
   unregister_chrdev(Major, DEVICE_NAME);            
   printk(KERN_INFO "Goodbye from the LKM!\n");
}

static int dev_open(struct inode *inodep, struct file *filep){
   numberOpens++;
   printk(KERN_INFO "Device has been opened %d time(s)\n", numberOpens);
   return 0;
}

static ssize_t dev_read(struct file *filep, char *buffer, size_t len, loff_t *offset){
   /*copy_to_user(buffer, message, size_of_message);
   return size_of_message;*/

	if((read_count+1 == write_ count) //buffer empty
		&& (filep->f_flags & O_NONBLOCK))
		return -EAGAIN;
	if(wait_event_interruptible(wq_read, (read_count+1 != write_ count)))
		return -ERESTARTSYS;
	int bytes_read = 0;
	while(bytes_read < len && (read_count+1 != write_ count) )
	{
		put_user(Buf[++read_count], buffer++);
		printk(KERN_INFO"read: readc %d, writec %d, buf[read] %c\n", read_count, write_ count, Buf[read_count]);
		bytes_read++;
		if(read_count == BUFFER_SIZE-1)
			read_count = -1;
	}
	wake_up_interruptible(&wq_write);
	
	printk(KERN_INFO"read: length %d, bytes %d\n", len, bytes_read);
	return bytes_read;
	
}

static ssize_t dev_write(struct file *filep, const char *buffer, size_t len, loff_t *offset){
   /*copy_from_user(message, buffer, size_of_message);
   message[size_of_message]=0;
   return size_of_message;*/
	
	if((read_count == write_ count) //buffer if full
		&&(filep->f_flags & O_NONBLOCK))
		return -EAGAIN;
	if(wait_event_interruptible(wq_write, (read_count != write_ count)))
		return -ERESTARTSYS;
	
	int bytes_written = 0;
	while( bytes_written < len && (write_ count != read_count) )
	{
		printk(KERN_INFO"write1: readc %d, writec %d\n", read_count, write_ count);
		get_user(Buf[write_ count++], buffer++);
		bytes_written++;
		if(write_ count == BUFFER_SIZE)
			count_write = 0;
		printk(KERN_INFO"write2: readc %d, writec %d\n", read_count, write_ count);
	}
	wake_up_interruptible(&wq_read);
	
	return bytes_written;
}

static int dev_release(struct inode *inodep, struct file *filep){
   numberOpens--;
   printk(KERN_INFO "Device successfully closed\n");
   return 0;
}

module_init(dev_init);
module_exit(dev_exit);